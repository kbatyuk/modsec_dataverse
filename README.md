# ModSecurity Dataverse Rules

Advanced ModSecurity WAF (Web Application Firewall) rules designed to protect Dataverse instances from automated threats, legacy clients, and resource exhaustion attacks.

**⚠️ Important**: This configuration must be implemented together with the [@kbatyuk/captcha](https://github.com/kbatyuk/captcha) PHP library for complete protection.

## Overview

This repository contains a comprehensive set of ModSecurity rules specifically tailored for Dataverse deployments. The rules are organized into four phases to effectively manage threats while maintaining service availability for legitimate users:

1. **Phase 1: Whitelist & Exemptions** - Trusted ranges and bypass rules
2. **Phase 2: Hard Drops** - Known malicious actors and suspicious patterns
3. **Phase 3: Soft Blocks** - Suspicious activity redirects to CAPTCHA
4. **Phase 4: Data Blocks** - Solr/database protection against resource exhaustion

## Features

- **Bot/Scraper Detection**: Blocks AI crawlers, scrapers, and automated tools
  - OpenAI, Meta, Alibaba, Baidu, Ahrefsbots, Amazon, and more
  - Customizable bot list for your environment

- **Legacy OS/Browser Filtering**: Redirects outdated clients to CAPTCHA
  - Windows 95/98/ME, old macOS (10.0-10.5)
  - Old iOS (1-12), Android (1-7)
  - Outdated Firefox, Chrome, Safari versions

- **Spoofer Detection**: Blocks requests with conflicting User-Agent headers
  - Detects Chrome/Firefox or Firefox/Chrome combinations

- **Resource Protection**: Prevents Solr/database abuse
  - Blocks excessive filter query depth (fq3+)
  - Query string length limits (>1000 chars)
  - Session fixation pattern detection (jsessionid in URI)

- **Intelligent Redirection**: Suspicious requests redirect to CAPTCHA challenge
  - Prevents false positive blocking of legitimate users
  - Sets `VerifiedHuman` cookie upon completion

- **Internal Network Trust**: Whitelists WHOI internal IP ranges
  - 128.128.0.0/16
  - 10.0.0.0/8

- **Google Bot Allowance**: Permits Googlebot and Google Inspection Tools
  - Ensures SEO crawling continues

## Installation

### Prerequisites

- Apache with ModSecurity module installed
- ModSecurity rule engine enabled
- Dataverse web application running on the protected server
- [@kbatyuk/captcha](https://github.com/kbatyuk/captcha) library deployed at `/captcha.php`

### Setup Steps

1. **Clone or download this repository**:
   ```bash
   git clone https://github.com/kbatyuk/modsec_dataverse.git
   ```

2. **Copy the rules file to your ModSecurity configuration**:
   ```bash
   cp modsecurity_localrules.conf /etc/modsecurity/rules/
   ```

3. **Include the rules in your main ModSecurity config** (`/etc/modsecurity/modsecurity.conf`):
   ```apache
   Include /etc/modsecurity/rules/modsecurity_localrules.conf
   ```

4. **Deploy the CAPTCHA library**:
   - Follow installation instructions at [@kbatyuk/captcha](https://github.com/kbatyuk/captcha)
   - Ensure it's accessible at `https://dataverse.whoi.edu/captcha.php`
   - The library handles token generation and validation

5. **Restart Apache** to apply the rules:
   ```bash
   sudo systemctl restart apache2
   # or
   sudo apachectl restart
   ```

6. **Test the configuration**:
   - Verify ModSecurity is active: `curl -I http://localhost`
   - Test with a known bot User-Agent: `curl -H "User-Agent: amazonbot" http://localhost`
   - Confirm CAPTCHA redirect works properly

## Configuration

### Customize Trusted Ranges

Edit the WHOI ranges in rule 1000:
```
SecRule REMOTE_ADDR "@ipMatch YOUR.IP.RANGE/16,10.0.0.0/8," \
    "id:1000,phase:1,nolog,allow,msg:'Internal Trusted Ranges'"
```

### Update Bot List

Modify the regex pattern in rule 1001 to add/remove bots:
```
SecRule REQUEST_HEADERS:User-Agent "@rx (turnitin|alibaba|crawler|...)" \
    "id:1001,phase:1,drop,status:403,..."
```

### Adjust Browser Version Thresholds

Change the version numbers in rules 2001-2005:
```
SecRule REQUEST_HEADERS:User-Agent "@rx Firefox/([1-9]|[1-9][0-9]|1[0-2][0-9])\." \
    "id:2001,..."
```

### Modify Solr Filter Limits

Adjust the filter depth threshold in rule 4000:
```
SecRule ARGS_NAMES "@rx fq[3-9]" \  # Change [3-9] to your desired limit
    "id:4000,..."
```

### Adjust Query String Length

Modify the length limit in rule 4003:
```
SecRule QUERY_STRING "@gt 1000" \  # Change 1000 to your threshold
    "id:4003,..."
```

### Change CAPTCHA Redirect URL

Replace all instances of:
```
https://dataverse.whoi.edu/captcha.php?orig_uri=%{REQUEST_URI}
```
with your actual CAPTCHA endpoint and Dataverse domain.

## Rule Details

### Phase 1: Whitelist & Exemptions (IDs: 1000-1007)

| ID | Purpose | Action |
|---|---|---|
| 1000 | Trust WHOI internal IP ranges | Allow |
| 1005 | Allow verified humans (cookie-based) | Allow |
| 1006 | Allow captcha.php requests | Allow |
| 1007 | Whitelist Google inspection tools | Allow |

### Phase 2: Hard Drops (IDs: 1001-1004)

| ID | Purpose | Action |
|---|---|---|
| 1001 | Block AI crawlers & scrapers (silent drop) | Drop (403) |
| 1002 | Block Frankenstein User-Agent spoofing | Deny (403) |
| 1003 | Block legacy Windows/macOS | Redirect to CAPTCHA |
| 1004 | Block legacy mobile OS | Redirect to CAPTCHA |

### Phase 3: Soft Blocks (IDs: 2001-2005)

| ID | Purpose | Action |
|---|---|---|
| 2001 | Redirect old Firefox browsers | Redirect to CAPTCHA |
| 2002 | Redirect old Chrome browsers | Redirect to CAPTCHA |
| 2003 | Redirect old Safari browsers | Redirect to CAPTCHA |
| 2004 | Redirect 32-bit Linux systems | Redirect to CAPTCHA |
| 2005 | Block suspicious .0.0.0 version patterns | Redirect to CAPTCHA |

### Phase 4: Data Blocks (IDs: 4000-4003)

| ID | Purpose | Action |
|---|---|---|
| 4000 | Block excessive Solr filter queries | Drop (403) |
| 4002 | Block jsessionid in URI (session fixation) | Redirect to CAPTCHA |
| 4003 | Block overly long query strings | Redirect to CAPTCHA |

## How It Works

### Redirect Flow

1. User request triggers a soft-block rule
2. User redirected to `captcha.php?orig_uri=<original_request>`
3. CAPTCHA library generates and validates token
4. Upon successful validation, `VerifiedHuman=1` cookie is set
5. User can access original resource (caught by rule 1005)

### Drop Flow

1. User request matches hard-block rule (known bot, suspicious pattern, etc.)
2. Connection silently dropped (HTTP 403)
3. Request logged for monitoring

## Logging

All rules generate logs in ModSecurity audit log:
```
/var/log/apache2/modsec_audit.log
```

View recent blocks:
```bash
tail -f /var/log/apache2/modsec_audit.log | grep -E "Dropped|Instant Ban|Redirect"
```

## Monitoring & Maintenance

### Check for False Positives

Monitor logs for legitimate traffic being blocked:
```bash
grep "Resource Protection" /var/log/apache2/modsec_audit.log
grep "Redirect Old" /var/log/apache2/modsec_audit.log
```

### Update Bot Lists

Review security advisories and update rule 1001 quarterly to catch new threats.

### Test Configuration Changes

Before applying to production:
```bash
# Test syntax
apachectl configtest

# Validate ModSecurity rules
modsec-rules-check /etc/modsecurity/rules/modsecurity_localrules.conf
```

## Troubleshooting

### Users Getting Redirected to CAPTCHA Unexpectedly

1. Check if User-Agent matches old browser patterns (rules 2001-2005)
2. Check query string length (rule 4003)
3. Verify `VerifiedHuman` cookie is being set properly by CAPTCHA library

### Legitimate Bots Being Blocked

1. Add whitelist exception in Phase 1 (before rule 1001)
2. Create a new rule with `allow` action:
   ```
   SecRule REQUEST_HEADERS:User-Agent "@rx yourbot" \
       "id:1099,phase:1,nolog,allow,msg:'Allow YourBot'"
   ```

### Performance Issues

1. Check if Solr is being attacked: `grep "Bloat Protection" /var/log/apache2/modsec_audit.log`
2. Review query string lengths: `grep "Query String Too Long" /var/log/apache2/modsec_audit.log`

### CAPTCHA Redirect Loop

Ensure rule 1006 allows `/captcha.php` and `/captcha-image.php` requests through without triggering other rules.

## Integration with @kbatyuk/captcha

This WAF configuration is designed to work seamlessly with the [@kbatyuk/captcha](https://github.com/kbatyuk/captcha) library:

- **Redirect Targets**: All soft-block rules redirect to the CAPTCHA challenge
- **Cookie Handling**: CAPTCHA library sets `VerifiedHuman=1` upon success (rule 1005 allows these)
- **Session Management**: Original URI is passed as query parameter for post-challenge redirect
- **Token Validation**: CAPTCHA library handles token generation and validation

Refer to the [@kbatyuk/captcha README](https://github.com/kbatyuk/captcha#readme) for deployment details.

## Security Considerations

1. **HTTPS Only**: Ensure all redirect URLs use HTTPS to prevent cookie theft
2. **Domain Validation**: Update all hardcoded `dataverse.whoi.edu` references to your actual domain
3. **Regular Updates**: Keep bot lists current as new threats emerge
4. **IP Whitelisting**: Consider adding additional trusted IP ranges for your organization
5. **Monitoring**: Set up alerts for unusual block patterns

## Performance Impact

- **CPU**: Minimal - regex matching is fast
- **Memory**: Negligible - rules are stateless
- **Latency**: <5ms per request for rule evaluation

## License

This project is open source and available under the MIT License.

## Contributing

Contributions are welcome! Please feel free to submit pull requests or open issues for:
- New bot patterns
- Rule improvements
- Bugfixes
- Documentation enhancements

## Support

For issues, questions, or suggestions:
- Open an issue on [GitHub](https://github.com/kbatyuk/modsec_dataverse/issues)
- Check [@kbatyuk/captcha](https://github.com/kbatyuk/captcha) for CAPTCHA-related questions

---

**Note**: This rule set is specifically designed for Dataverse instances and may require adaptation for other applications. Always test in a non-production environment first.
