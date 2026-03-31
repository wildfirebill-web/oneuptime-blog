# How to Build a Session Store in Go with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Go, Session

Description: Learn how to build a Redis-backed session store in Go covering session creation, retrieval, refresh, and invalidation using go-redis.

---

Storing sessions in Redis instead of memory or cookies enables horizontal scaling. Any server instance can read session data because it is all stored centrally in Redis. Sessions are stored as hashes keyed by a random token with automatic TTL expiration.

## Session Store Implementation

```go
package session

import (
    "context"
    "crypto/rand"
    "encoding/hex"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

const sessionTTL = 2 * time.Hour
const sessionPrefix = "session:"

type Store struct {
    rdb *redis.Client
}

func NewStore(rdb *redis.Client) *Store {
    return &Store{rdb: rdb}
}

func generateToken() (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return hex.EncodeToString(b), nil
}

func (s *Store) Create(ctx context.Context, userID string, data map[string]string) (string, error) {
    token, err := generateToken()
    if err != nil {
        return "", err
    }

    key := sessionPrefix + token
    fields := make([]interface{}, 0, len(data)*2+2)
    fields = append(fields, "userID", userID)
    for k, v := range data {
        fields = append(fields, k, v)
    }

    pipe := s.rdb.Pipeline()
    pipe.HSet(ctx, key, fields...)
    pipe.Expire(ctx, key, sessionTTL)
    _, err = pipe.Exec(ctx)
    if err != nil {
        return "", fmt.Errorf("create session: %w", err)
    }

    return token, nil
}

func (s *Store) Get(ctx context.Context, token string) (map[string]string, error) {
    result, err := s.rdb.HGetAll(ctx, sessionPrefix+token).Result()
    if err != nil {
        return nil, err
    }
    if len(result) == 0 {
        return nil, nil // session not found or expired
    }
    return result, nil
}

func (s *Store) Refresh(ctx context.Context, token string) error {
    return s.rdb.Expire(ctx, sessionPrefix+token, sessionTTL).Err()
}

func (s *Store) Set(ctx context.Context, token, field, value string) error {
    key := sessionPrefix + token
    pipe := s.rdb.Pipeline()
    pipe.HSet(ctx, key, field, value)
    pipe.Expire(ctx, key, sessionTTL)
    _, err := pipe.Exec(ctx)
    return err
}

func (s *Store) Destroy(ctx context.Context, token string) error {
    return s.rdb.Del(ctx, sessionPrefix+token).Err()
}

func (s *Store) Exists(ctx context.Context, token string) (bool, error) {
    n, err := s.rdb.Exists(ctx, sessionPrefix+token).Result()
    return n > 0, err
}
```

## HTTP Middleware

```go
func SessionMiddleware(store *Store) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            cookie, err := r.Cookie("SESSION")
            if err != nil || cookie.Value == "" {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }

            session, err := store.Get(r.Context(), cookie.Value)
            if err != nil || session == nil {
                http.Error(w, "Session expired", http.StatusUnauthorized)
                return
            }

            store.Refresh(r.Context(), cookie.Value)

            ctx := context.WithValue(r.Context(), "userID", session["userID"])
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

## Login and Logout Handlers

```go
func LoginHandler(store *Store) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Verify credentials (omitted)
        userID := "user:42"
        data := map[string]string{"role": "admin", "name": "Alice"}

        token, err := store.Create(r.Context(), userID, data)
        if err != nil {
            http.Error(w, "Internal error", 500)
            return
        }

        http.SetCookie(w, &http.Cookie{
            Name:     "SESSION",
            Value:    token,
            HttpOnly: true,
            Secure:   true,
            Path:     "/",
            MaxAge:   int(sessionTTL.Seconds()),
        })
        w.WriteHeader(http.StatusOK)
    }
}

func LogoutHandler(store *Store) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if cookie, err := r.Cookie("SESSION"); err == nil {
            store.Destroy(r.Context(), cookie.Value)
        }
        http.SetCookie(w, &http.Cookie{Name: "SESSION", MaxAge: -1})
        w.WriteHeader(http.StatusNoContent)
    }
}
```

## Summary

A Redis session store in Go uses hashes keyed by a random token with TTL expiration. Creating a session involves generating a secure random token, storing user fields as a hash, and setting an expiry. HTTP middleware validates the cookie token against Redis and refreshes the TTL on each request. Destroying a session deletes the key immediately rather than waiting for expiry.
