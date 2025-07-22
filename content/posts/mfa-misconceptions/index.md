---
title: "The Importance of MFA and some misconceptions"
date: 2025-07-22T20:57:48+02:00
draft: false
toc: false
tags: ["Cybersecurity", "Authentication", "MFA"]
---
When discussing the security of information systems, one of the foundational concepts is **authentication**, which can be defined as the process of verifying the identity of a user, device, or system attempting to access a resource. Authentication answers the question: *“Who are you?”*, and it does so by requiring evidence of identity.

This verification is necessary because, much like in physical security, access to protected spaces or resources must be controlled based on evidence of identity. In the physical world, guards at a secure facility do not simply allow anyone to enter; they check badges, keys, or biometric features, and may even recognize familiar individuals. The possession of a key or ID, knowledge of a code, or inherent physical traits serve as tangible proofs of identity. Similarly, in digital systems, where there is no direct way for the system to “see” or “sense” the claimant, we rely on analogous mechanisms: transmitted credentials and controlled challenges that emulate the same principles of trust established at a physical checkpoint.

### The Three Classic Authentication Factors

Historically, the framework for authentication has been formalized in terms of **factors of authentication**, which fall into three mutually distinct categories, often summarized as:

* **Something you know** — a secret memorized and presented at the time of authentication, such as a password, PIN, or the answer to a security question. This is the oldest and most common factor, dating back to the earliest computer systems, where login prompts asked users to type a password. This factor is inexpensive to implement and does not require specialized hardware, but it suffers from significant weaknesses: secrets can be guessed, stolen, reused, or intercepted, and humans tend to choose weak or repetitive passwords for convenience.  

* **Something you have** — a physical or digital token in your possession, such as a smart card, a time-based one-time password (TOTP) generator, a mobile app, or a hardware security key (e.g., FIDO2). The assumption here is that the token is difficult to duplicate and remains under the owner’s control. This factor mitigates some weaknesses of knowledge-based authentication by adding a physical element, but it introduces operational challenges: tokens can be lost, damaged, or stolen, and managing them at scale requires additional logistics and infrastructure.  

* **Something you are** — an intrinsic characteristic of your body, or **biometric evidence**, such as a fingerprint, retinal or iris pattern, facial geometry, voice pattern, or even behavioral traits (like typing rhythm or gait). Biometric authentication is compelling because it is hard to forge and convenient for users. However, it also raises concerns about privacy, data permanence (you cannot change your fingerprint if it is compromised), and accuracy (with risks of false positives and false negatives).

These three factors are considered **orthogonal**, meaning that each addresses different kinds of risk and they are stronger when combined. This is the basis of **multi-factor authentication (MFA)**, which requires the claimant to present evidence from at least two different categories (e.g., a password and a hardware token).

### Historical Context and Formalization

The three-factor model has its roots in the late 20th century, when computer systems and networks began handling more sensitive and valuable information. Mainframe systems of the 1960s and 1970s were already using password-based mechanisms, but as networks expanded and threats increased, the need for stronger assurances became evident.

In particular, the U.S. Department of Defense’s *Orange Book* (Trusted Computer System Evaluation Criteria, TCSEC, 1983\) emphasized the importance of identity verification in secure systems. Later, the National Institute of Standards and Technology (NIST) published a series of guidelines (notably **NIST SP 800-63**) that formalized identity assurance levels (IAL), authenticator assurance levels (AAL), and recommended the use of multiple factors for high-risk scenarios. These guidelines have since influenced international standards and commercial best practices.

### Authentication vs. Authorization

It is also critical to distinguish **authentication** from **authorization**, two related but distinct components of access control.

* **Authentication** confirms *who you are*: it is the process of establishing the identity of the entity interacting with the system.  
* **Authorization** determines *what you are allowed to do*: it governs the permissions and access rights granted to an authenticated entity.

Misunderstanding this distinction can lead to design flaws, where systems incorrectly assume that knowing *who* the user is automatically confers the right to perform certain actions, a fallacy that can open the door to privilege escalation and other attacks.

It is understandable to see professionals struggle with the distinction between authentication and authorization, especially when widely used standards themselves adopt misleading terminology. A notable example is the HTTP status code `401 Unauthorized`, which seems to imply an authorization failure. However, RFC 7235 clarifies that it actually indicates the absence of valid authentication credentials, in other words, the client is unauthenticated. Conversely, if the client is authenticated but lacks permission to access the resource, the appropriate status is `403 Forbidden`. Such ambiguous terminology can contribute to confusion, leading to poor error handling, security misconfigurations, and misleading diagnostics in practice.

Below is an expanded and more technically detailed version of your draft, preserving your structure and style while deepening the discussion:

---

## *The Importance of Multi-Factor Authentication*

### What is “strong” authentication?

The term **strong authentication** refers to an authentication mechanism that significantly raises the barrier against unauthorized access by requiring evidence from *multiple independent factors*, each of which presents distinct challenges to an attacker.

According to NIST SP 800-63B (“Digital Identity Guidelines”), strong authentication is achieved when authentication combines **two or more different factors**.

This taxonomy is rooted in the observation that each factor type involves a distinct trust assumption and failure mode. For example, a password can be guessed or leaked; a smartcard can be stolen; a fingerprint could theoretically be replicated, but achieving all three simultaneously is much more difficult.

This principle is also emphasized by regulatory frameworks such as the European Union’s **Revised Payment Services Directive (PSD2)**, specifically its provision on **Strong Customer Authentication (SCA)**, which mandates that authentication in financial services “validates the identity of the user through at least two factors which are independent from each other, and designed in such a way that the breach of one does not compromise the reliability of the other”.

The emphasis on **independence** is critical: if compromising one factor gives an attacker a pathway to compromise the second (e.g., a password reset sent to email when both accounts use the same password), the overall scheme fails to meet the standard of strong authentication.

### Why does strong authentication matter?

Weak or single-factor authentication remains one of the most commonly exploited vulnerabilities in both consumer and enterprise environments.

For instance, the **Verizon Data Breach Investigations Report (DBIR)** consistently finds that over 80% of hacking-related breaches involve stolen, weak, or reused credentials. Similarly, Microsoft reports that enabling MFA can prevent over 99.9% of automated account compromise attacks, a figure reflecting the asymmetry between the effort needed to phish or guess a password and the effort needed to also obtain a second independent factor.

More granular studies (including Google’s analysis of millions of accounts) demonstrate that even lower-assurance factors, like SMS OTPs, reduce bulk phishing attack success by over 96%, and targeted attacks by over 76%. These figures improve further when higher-assurance factors (e.g., hardware security keys) are used.

The underlying reason is that MFA introduces a requirement that attackers compromise at least two unrelated mechanisms simultaneously: for example, intercepting a password *and* stealing a device or defeating a biometric reader. This drastically increases both the cost and complexity of attacks, often forcing attackers to move on to softer targets.

It also effectively neutralizes whole classes of attacks, such as credential stuffing and password spraying, which rely on automated exploitation of password reuse.

### Common misconceptions about MFA

Despite its benefits, MFA is often misunderstood or improperly implemented, sometimes resulting in a false sense of security.

### MFA vs. multi-step authentication

A frequent mistake is conflating **multi-step** authentication with **multi-factor** authentication. For example, many services send a **one-time code (OTC)** to a user’s email or SMS after password entry. While this adds an additional *step*, it does not necessarily add a true *factor*, because both steps may belong to the same category of authentication (typically *something you know* or *something you control digitally*).

Consider the following example:

* Step 1: Enter a password (something you know).  
* Step 2: Retrieve a code from email (also something you know and control via another password).

Here, if the attacker compromises the email account, which may also be secured by the same password or an easily guessed one, both factors fall simultaneously, invalidating the intended security benefit.

For authentication to qualify as MFA, each factor must be drawn from a distinct category, and their compromise must be independent. Proper examples include:

* Password \+ hardware security token (knowledge \+ possession)  
* Password \+ fingerprint (knowledge \+ inherence)

### The problem with SMS and email OTPs

Another misconception is that **SMS or email-based OTCs are inherently secure** because they add a step after password entry. While they are better than no additional step, they suffer from serious weaknesses:

* SMS is vulnerable to SIM-swapping, SS7 protocol attacks, and social engineering of mobile carriers.  
* Email can be compromised via phishing, password reuse, or malware, and is often the recovery vector for many accounts, making it a high-value target.

A well-documented example of this is the scam that targeted **O2 customers** in the UK, where attackers used social engineering to extract SMS OTCs under the guise of a prize verification process. Once customers revealed the codes, attackers immediately accessed and took over the accounts.

### Biometric fallacies

Even biometric factors (*something you are*) are sometimes misused or misunderstood. People often believe that biometrics are foolproof. However:

* Many biometric systems are based on templates that can be stolen and replayed.  
* Some can be spoofed with photographs, silicone molds, or deepfake techniques.  
* Unlike passwords, compromised biometric data cannot be changed.

Therefore, biometrics are best used as *one factor among others*, not as a standalone mechanism.

### Recovery paths can undermine MFA

Even properly designed MFA can be defeated if account recovery paths bypass it. For example, if a service allows a password reset via email without re-verifying possession of the second factor, an attacker can compromise the email account and reset the password, effectively bypassing MFA.

---

## *Best Practices for Stronger Authentication*

Multi-factor authentication (MFA) has proven itself to be one of the most effective countermeasures against unauthorized access, credential stuffing, and account takeovers. However, its effectiveness depends critically on how it is implemented and understood.

In the preceding sections, we examined the foundational principles of authentication, clarified the distinction between authentication and authorization, and addressed common misconceptions about what constitutes “strong” authentication. Misapplied MFA, such as using two steps from the same factor category, or relying on insecure channels like SMS or email, can create a false sense of security.

### Recommendations for implementing MFA effectively

To close, here are several best practices based on guidance from NIST, ENISA, and industry experience:

* **Ensure factor independence**: Select factors from different categories (*know*, *have*, *are*) and ensure compromise of one does not compromise the others.  

* **Prefer phishing-resistant methods**: Whenever possible, opt for cryptographically bound hardware authenticators (e.g., FIDO2/WebAuthn security keys) or platform-based authenticators (e.g., Windows Hello, Touch ID) that resist phishing and man-in-the-middle attacks.  

* **Avoid SMS and email as primary second factors**: Use these channels only as fallback or recovery mechanisms, not as primary authentication factors, due to their susceptibility to interception, SIM swapping, and social engineering.  

* **Educate users about social engineering**: Many attacks bypass MFA by tricking users into revealing one-time codes or approving fraudulent login attempts. Training users to recognize and resist such attempts is essential.  

* **Monitor and adapt**: Treat authentication as a living component of your security architecture. Monitor for evolving threats, evaluate the security of chosen factors periodically, and adapt policies as needed.

### Final thoughts

Authentication is more than a technical hurdle; it is a cornerstone of trust in digital systems. Understanding the nuances of MFA, resisting the allure of superficial “two-step” schemes, and adopting strong, independent factors are essential steps toward robust access control.

By grounding your implementation in the principles outlined here, independence of factors, resistance to phishing, and resilience against social engineering, you can significantly reduce the risk of unauthorized access while preserving usability.
