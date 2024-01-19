<!-- # PKI in the Home(lab) -->
# Self-Hosted Certificate Authority and Mutual TLS with step-ca and YubiKeys
## Implementation Overview
- self-hosted CA on RPI with `step-ca``
    - Yubikey as low cost HSM, device attestation for my other Yubis
    - remote clients over Tailscale
- Caddy Web Server as ACME client
    - 24h certificate validity, automatic renewal, JIT issuance
    - mutual TLS with Yubikeys 
    - reference good integration with Docker and Tailscale

## My Journey
I've been a consumer of certificates issued by public authorities (CAs) for years. I requested my first certificate from Comodo way back in the 2010's for email encryption and signing with GPG. Once [Let's Encrypt's ACME service](https://letsencrypt.org/how-it-works/) became widely known, I started using them for my certificate needs. 

There are a some consideratoins with using a system like Let's Encrypt.

### Pros
- free as in speech and free as in beer ([libre and gratis](https://www.gnu.org/philosophy/free-sw.en.html))
- certificates are trusted automatically by practically everyone
- ACME protocol and tools like [`certbot`](https://certbot.eff.org/) make implementation easy on many systems
- excellent documentation, community adoption and support
- certificates are valid for 90 days (forcing you to automate)

### Cons
- certificates are valid for 90 days (no flexibility)
- public certificates are public knowledge ([certificate transparency](https://www.mayhem.security/blog/certificate-transparency-does-more-harm-than-good-heres-why))
- requires an internet-accessible web server (HTTP/TLS challenges) or storing credentials for your DNS on/near your web server 

## Modern Problems
Security is often held in opposition to usablity and certificates can certainly embody this. Traditionally, certificates are valid for a year or two at a time because rotating them is a pain and potentially [very disruptive](https://www.zdnet.com/article/government-shutdown-tls-certificates-not-renewed-many-websites-are-down/). 

But once you stop using one, how do you tell the world that they should no longer trust an otherwise valid certificate? [Smallstep has a great blog post](https://smallstep.com/blog/passive-revocation/) but, in brief, humans developed several ways to solve the problem but none are ideal.

<u>Certificate Revocation Lists (CRLs)</u>: <br>
 A list of *all* certificates which a CA has issued and then revoked. 
 - can get very long, causing performance issues
 - effectiveness depends on clients to check the CRL and most apps do not enforce validity
 - CRLs may only be updated weekly or less

<u>Online Certificate Status Protocol (OSCP)</u>: <br>
An API through which clients can query an authority for the status of a certificate. 
- still requires clients to actively verify status
- adds a connection, and thus latency, to every web connection

The answer is to instead issue very short-lived certificates and eliminate the need for revocation mechanisms entirely. Smallstep calls this *passive revocation*.

## Modern Solutions
So how do we achieve all the pros while avoiding the cons? Just run your own certificate authority! Enterprise administrators will be familiar with Active Directory Certificate Services but... that won't run on a spare Raspberry Pi (red X icon), needs expensive licenses(red X icon), and isn't open source (red X icon). 

Enter `step-ca`: https://github.com/smallstep/certificates
- configurable certificate lifetimes (24h by default)
- certificates for your domain - without public record
- convenience of automated issuance via ACME
- greater security through passive revocation
- the smug satisfaction of an over-engineered solution

## Detailed Implementation
Coming soon!

## Future Directions
These might be diminishing returns since I have basically met these needs in other ways.
- use SSH certificates rather than keys 
- put a CA on the public internet. Uses? Github Actions certificates rather than hardcoded creds. What else?...
