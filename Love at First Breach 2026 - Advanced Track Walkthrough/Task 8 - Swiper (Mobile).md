# Swiper - Love at First Breach 2026 - Advanced Track

## Challenge Overview

Swiper is a mobile application challenge where we analyze an Android APK to find vulnerabilities and extract sensitive information. The app is a "love quote sharing" platform where users can find people based on 5-word descriptions and share love quotes.

## APK Information

- **APK Location**: `app-debug-1771154850568.apk`
- **Package Name**: `com.swiper`
- **App Type**: Android (Jetpack Compose)
- **API Base URL**: `https://10-81-118-246.reverse-proxy.cell-prod-eu-west-1b.vm.tryhackme.com/`

## Static Analysis

### Decompiling the APK

Using `apktool` to decompile the APK:

```bash
apktool d app-debug-1771154850568.apk -o decompiled
```

### Key Findings from Smali Code Analysis

The API service file (`ApiService.smali`) reveals multiple endpoints:
- `login` - User authentication
- `register` - User registration
- `discover` - Find profiles
- `getProfile` / `getUserProfile` - Profile information
- `getQuotes` / `getUserPublicQuotes` - Quote endpoints
- `createQuote` / `deleteQuote` - Quote management
- `getMembership` - Premium membership info
- `enable2fa` / `disable2fa` - Two-factor authentication
- And more...

### API Base URL Discovery

Found in `RetrofitClient.smali`:
```
BASE_URL = "https://10-81-118-246.reverse-proxy.cell-prod-eu-west-1b.vm.tryhackme.com/"
```

## Questions & Answers

### Question 1: Get a premium membership. What's your premium subscription ID?

**Answer**: `THM{is_this_what_freemium_means?}`

### Question 2: Figure out who is the user whose email is `shadow777@thm.thm`. What is their password?

**Answer**: Password is `loveangel`

### Question 3: Login as the user you found on the previous question. What's the flag in their private quotes?

**Answer**: `THM{ch4ll3ng3_sw1p3d_d0wn}`

## Exploitation Techniques

### 1. Finding Premium Features

By grepping for "premium" in the decompiled APK:
```bash
grep -r "premium" /tmp/decompiled/smali_classes5/
```

This reveals the membership-related endpoints and premium features.

### 2. SQL Injection for User Discovery

Since premium features require authentication, we can use multi-threaded bot scripts to:
- Create multiple accounts
- Perform SQL injection on the registration/login endpoints
- Find the user with email `shadow777@thm.thm`

### 3. API Endpoint Analysis

Key endpoints discovered:
- `/api/auth/login` - Login
- `/api/auth/register` - Registration  
- `/api/profiles/discover` - Profile discovery
- `/api/quotes` - Quote management
- `/api/membership` - Membership info
- `/api/2fa/enable` - 2FA enable

### 4. Device Fingerprint Bypass

The app requires a device fingerprint for certain operations. This can be extracted from the app's authentication interceptor code.

## Key Takeaways

1. **Static Analysis is Crucial**: Even obfuscated mobile apps leak API endpoints and URLs in smali code
2. **API Security**: Never rely on client-side controls for premium features
3. **Authentication**: Device fingerprinting can be bypassed; server-side validation is essential
4. **Rate Limiting**: The app has rate limits that can be bypassed with multiple threads

## Tools Used

- `apktool` - APK decompilation
- `strings` - Binary analysis
- `grep` - Pattern matching
- Python scripts for automation
- `curl` - API interaction

---
