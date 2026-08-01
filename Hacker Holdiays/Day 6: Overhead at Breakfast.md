# OSINT Challenge Walkthrough: "Concierge Briefing"

## 🎯 Objective
The goal of this challenge is to identify a hidden online account based on clues found in a leaked conversation screenshot from a hotel breakfast terrace, and subsequently retrieve a hidden flag.

## 🛠️ Tools Used
*   **Image Viewer**: For detailed inspection of the provided screenshot.
*   **Gravatar Email Checker**: To verify account existence via email address (`://gravatar.com`).
*   **CyberChef**: A web-based tool for decoding data formats (specifically Base64).

---

## 📝 Step-by-Step Methodology

### Phase 1: Intelligence Gathering (Image Analysis)
The first step involves a thorough examination of the provided conversation screenshot. The objective is to extract **Personally Identifiable Information (PII)** or unique identifiers.

1.  **Contextual Review**: The scenario describes a guest overhearing a conversation. This implies the clues are textual within the chat bubbles.
2.  **Data Extraction**: Upon inspecting the image, a user named "Lambo!" mentions a specific contact method and a hint about a platform:
    *   **Email Address**: `lambobytelotushotel@gmail.com` (Derived from the visible text `lambobyte lotushotel@gmail.com`, removing the space to form a valid email structure).
    *   **Platform Hint**: "Started with a G" and "free tool that let me upload my profile."

![OSINT Clue Screenshot](Media/Screenshot%202026-08-01%20220010.png)   


### Phase 2: Hypothesis & Enumeration
Based on the extracted clues, we formulate a hypothesis to locate the target account.

*   **Hypothesis**: The "G" platform is likely **GitHub** or **Gravatar**.
*   **Action 1 (GitHub)**: A search for the username `lambobyte` on GitHub yields no relevant results matching the context.
*   **Action 2 (Gravatar)**: Since Gravatar profiles are indexed by **email address** rather than just username, this is the primary vector.
    *   Navigate to `https://://gravatar.com`.
    *   Input the extracted email: `lambobytelotushotel@gmail.com`.
    *   **Result**: A profile is successfully located.

![Gravatar Result](Media/Screenshot%202026-08-01%20220735.png)

### Phase 3: Data Retrieval
Accessing the identified Gravatar profile reveals the critical payload.

*   **Observation**: The profile bio or display name contains a non-human-readable string:
    `VEhNe1MzY3JIVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9`
*   **Analysis**: The string structure (alphanumeric characters ending with `=`) is characteristic of **Base64 encoding**, a common method for obscuring flags in Capture The Flag (CTF) challenges.

![Base64 Hash](Media/Screenshot%202026-08-01%20220941.png)

### Phase 4: Decoding & Flag Extraction
The final step is to decode the obscured string to reveal the solution.

1.  **Tool Selection**: Use **CyberChef** (or any Base64 decoder).
2.  **Operation**: Select the "From Base64" operation.
3.  **Execution**: Input the hash `VEhNe1MzY3JIVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9`.
4.  **Output**: The decoder returns the plaintext flag.

---

## 🏁 Final Solution

The decoded flag is:

```text
THM{S3crHT_Pr0fil3_H4s_b33n_Ident1fi3d}
```

## 💡 Key Learnings
*   **Email Correlation**: Email addresses are unique identifiers that can link identities across different platforms (e.g., from a chat log to a Gravatar profile).
*   **Platform Nuances**: Different platforms index users differently (Usernames vs. Email Hashes).
*   **Encoding Recognition**: Identifying common encoding schemes like Base64 is essential for retrieving hidden data in OSINT and CTF scenarios.
