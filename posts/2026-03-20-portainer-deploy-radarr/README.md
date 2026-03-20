# How to Deploy Radarr via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Radarr, Movies, Media Management, Self-Hosted

Description: Deploy Radarr via Portainer as an automatic movie collection manager that downloads, organizes, and upgrades your movie library quality automatically.

## Introduction

Radarr is the movie-focused counterpart to Sonarr. It monitors your movie wishlist, automatically downloads when items become available, and can upgrade to better quality versions as they appear. Deploying via Portainer makes it easy to manage alongside Sonarr and other media tools.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - radarr_config:/config
      - /mnt/media/movies:/movies
      - /mnt/media/downloads:/downloads
    ports:
      - "7878:7878"
    restart: unless-stopped

volumes:
  radarr_config:
```

## Configuring Radarr

### Quality Profiles

Navigate to **Settings > Profiles** to configure quality tiers:

- **HD-1080p**: 1080p WEB-DL, Blu-ray
- **Ultra-HD**: 4K UHD, 4K Remux
- **Any**: Accept any quality available

### Connecting to Download Client

1. Navigate to **Settings > Download Clients > Add**
2. Select qBittorrent or your preferred client
3. Configure host, port, and credentials
4. Set category: `radarr`

### Adding Movies

1. Click **Add New** and search for a movie
2. Select the movie from results
3. Set root folder: `/movies`
4. Set quality profile
5. Click **Add Movie**

For movies already in your library:
1. **Add New > Browse existing movies in library**

## Movie List Integration

Import movies from Trakt.tv, IMDb lists, or other sources:

1. Navigate to **Settings > Import Lists**
2. Add a Trakt list (requires Trakt authentication)
3. Radarr will automatically add movies from your watchlist

## Custom Naming Format

In **Settings > Media Management > Movie Naming**:

```
{Movie Title} ({Release Year}) [IMDB-{ImdbId}] {Edition Tags} {Quality Full}
```

## Quality Cutoff and Upgrades

Configure Radarr to upgrade quality automatically:

1. In your quality profile, set the **Cutoff** (e.g., Blu-ray 1080p)
2. Enable **Upgrade Until** quality
3. Radarr will download better versions until the cutoff is met

## Radarr API Integration

```bash
# Get all movies
curl "http://localhost:7878/api/v3/movie" \
  -H "X-Api-Key: your_radarr_api_key"

# Add a movie
curl -X POST "http://localhost:7878/api/v3/movie" \
  -H "X-Api-Key: your_radarr_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "The Dark Knight",
    "tmdbId": 155,
    "qualityProfileId": 1,
    "rootFolderPath": "/movies",
    "monitored": true,
    "addOptions": {"searchForMovie": true}
  }'
```

## Conclusion

Radarr deployed via Portainer automates your movie collection with quality management and automatic upgrades. Combined with Prowlarr for indexers and a download client, it handles the entire movie acquisition pipeline. The API makes it easy to integrate with custom scripts for adding movies from external sources like watchlists or recommendation systems.
