# How to Validate IPv4 Addresses Using Regex in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, Regex, IPv4, Validation, Patterns, Networking

Description: Learn how to validate IPv4 address strings using regular expressions in Java, with a strict pattern that correctly enforces the 0-255 octet range and rejects leading zeros.

## Simple Pattern (Not Sufficient)

```java
import java.util.regex.Pattern;

// Naive: accepts format but not octet range
Pattern naive = Pattern.compile("^\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}$");

System.out.println(naive.matcher("192.168.1.1").matches());    // true
System.out.println(naive.matcher("999.999.999.999").matches()); // true - WRONG
```

## Strict Regex (Validates 0-255)

```java
import java.util.regex.Pattern;

public class Ipv4Validator {

    // Each alternate branch handles a sub-range:
    // 25[0-5]  → 250-255
    // 2[0-4]\d → 200-249
    // 1\d{2}   → 100-199
    // [1-9]\d  → 10-99
    // \d       → 0-9
    private static final String OCTET =
        "(?:25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]\\d|\\d)";

    private static final Pattern STRICT_IPV4 =
        Pattern.compile("^" + OCTET + "(?:\\." + OCTET + "){3}$");

    public static boolean isValidIPv4(String s) {
        if (s == null) return false;
        return STRICT_IPV4.matcher(s).matches();
    }

    public static void main(String[] args) {
        String[][] tests = {
            {"192.168.1.1",       "true"},
            {"0.0.0.0",           "true"},
            {"255.255.255.255",   "true"},
            {"256.0.0.1",         "false"},   // Octet > 255
            {"192.168.1",         "false"},   // Missing octet
            {"192.168.1.1.1",     "false"},   // Extra octet
            {"192.168.01.1",      "false"},   // Leading zero
            {"::1",               "false"},   // IPv6
            {"",                  "false"},   // Empty
            {" 192.168.1.1",      "false"},   // Leading space
        };

        for (String[] tc : tests) {
            boolean result = isValidIPv4(tc[0]);
            boolean expected = Boolean.parseBoolean(tc[1]);
            String status = result == expected ? "PASS" : "FAIL";
            System.out.printf("[%s] %-25s -> %s%n", status, tc[0], result);
        }
    }
}
```

## Using Pattern as a Compiled Constant

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public final class IpPatterns {

    public static final String IPV4_REGEX =
        "^(?:25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]\\d|\\d)" +
        "(?:\\.(?:25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]\\d|\\d)){3}$";

    // Compile once - Pattern is thread-safe
    public static final Pattern IPV4_PATTERN = Pattern.compile(IPV4_REGEX);

    private IpPatterns() {}

    public static boolean isIPv4(String address) {
        return address != null && IPV4_PATTERN.matcher(address).matches();
    }
}
```

## Extracting IPs from Text (Log Parsing)

```java
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class IpExtractor {

    private static final String OCTET =
        "(?:25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]\\d|\\d)";

    // Non-anchored with word boundaries to find IPs inside larger text
    private static final Pattern FINDER = Pattern.compile(
        "\\b" + OCTET + "(?:\\." + OCTET + "){3}\\b"
    );

    public static List<String> extractIPs(String text) {
        List<String> ips = new ArrayList<>();
        Matcher m = FINDER.matcher(text);
        while (m.find()) {
            ips.add(m.group());
        }
        return ips;
    }

    public static void main(String[] args) {
        String log = "[2026-03-20] Request from 192.168.1.50 forwarded to 10.0.0.1";
        System.out.println(extractIPs(log));  // [192.168.1.50, 10.0.0.1]
    }
}
```

## Bean Validation Annotation (Jakarta)

```java
import jakarta.validation.Constraint;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import jakarta.validation.Payload;
import java.lang.annotation.*;
import java.util.regex.Pattern;

@Documented
@Constraint(validatedBy = ValidIPv4.Validator.class)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidIPv4 {
    String message() default "Invalid IPv4 address";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};

    class Validator implements ConstraintValidator<ValidIPv4, String> {
        private static final Pattern P = Pattern.compile(
            "^(?:25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]\\d|\\d)" +
            "(?:\\.(?:25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]\\d|\\d)){3}$"
        );

        @Override
        public boolean isValid(String value, ConstraintValidatorContext ctx) {
            if (value == null) return false;
            return P.matcher(value).matches();
        }
    }
}

// Usage on a DTO field:
// @ValidIPv4
// private String clientIp;
```

## Conclusion

Compile the `Pattern` once as a `static final` field - `Pattern` is thread-safe and reuse is free. The strict octet alternation `(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)` correctly encodes the 0-255 range and rejects leading zeros. For log parsing, drop the `^$` anchors and add word boundaries. In Spring/Jakarta applications, wrap the pattern in a custom `ConstraintValidator` for declarative field-level validation.
