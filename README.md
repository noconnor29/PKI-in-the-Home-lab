<!-- # PKI in the Home(lab) -->
# Self-Hosted Certificate Authority and Mutual TLS with step-ca and YubiKeys
## Table of Contents
1. [My Journey in PKI](#my-journey-in-pki)
2. [Modern Problems](#modern-problems)
3. [Require Modern Solutions](#require-modern-solutions)
4. [Implementation Overview](#implementation-overview)
5. [Future Directions](#future-directions)

## My Journey in PKI
As a long-time consumer of certificates from public authorities (CAs), I've navigated the challenges of securing my communications since the early 2010s. I began with Comodo and later transitioned to [Let's Encrypt's ACME service](https://letsencrypt.org/how-it-works/) for its versatility and wide support.

However, relying solely on public CAs comes with its set of challenges and limitations.

#### Pros 👍
- [Free as in speech and free as in beer ](https://www.gnu.org/philosophy/free-sw.en.html) - whichever is more important to you!
- Certificates are trusted automatically by practically everyone
- ACME protocol and tools like [`certbot`](https://certbot.eff.org/) make implementation easy on many systems
- Excellent documentation, community adoption and support
- Certificates are valid for 90 days (automate!)

#### Cons 👎
- certificates are valid for 90 days (no flexibility)
- Public certificates are public knowledge ([certificate transparency](https://www.mayhem.security/blog/certificate-transparency-does-more-harm-than-good-heres-why))
- Requires an internet-accessible web server or storing credentials for your DNS on your web server 

## Modern Problems...
Security and ease of use are often opposing goals and certificates can certainly embody this. Traditionally, certificates are valid for a year or two at a time because rotating them is a pain and potentially [very disruptive](https://www.zdnet.com/article/government-shutdown-tls-certificates-not-renewed-many-websites-are-down/).

But once you *stop* using one, how do you tell the world that they should no longer trust an otherwise valid certificate? Smallstep has a [great blog post](https://smallstep.com/blog/passive-revocation/) exploring this problem. In summary, smart people have come up with several ways approaches but none are ideal.

The two primary solutions, Certificate Revocation Lists (CRLs) and Online Certificate Status Protocol (OSCP), rely on clients to actively check validity and can cause performance issues in certain contexts.

By embracing very short-lived certificates, we can eliminate the need for revocation mechanisms entirely. Smallstep calls this *passive revocation*.

## ...Require Modern Solutions
So how do we achieve all pros while avoiding the cons? Just r!un your own certificate authority! 

Enterprise admins may be familiar with Active Directory Certificate Services but that<br>
&nbsp;&nbsp;&nbsp;&nbsp;❌ won't run on a spare Raspberry Pi<br>
&nbsp;&nbsp;&nbsp;&nbsp;❌ needs expensive licenses<br>
&nbsp;&nbsp;&nbsp;&nbsp;❌ isn't open source<br>

Enter `step-ca`[<img src="https://github.githubassets.com/images/modules/logos_page/GitHub-Mark.png" alt="GitHub Logo" width="20" height="20">](https://github.com/smallstep/certificates)
- configurable certificate lifetimes (default: 24h)
- certificates for your domain - without public record
- convenience of automated issuance via ACME
- greater security through passive revocation
- the smug satisfaction of an over-engineered solution 😎

## Implementation Overview
I ended up with a relatively minimal setup to meet my needs: 
- self-hosted CA on Raspberry Pi with `step-ca`
    - Yubikey 5 as low cost HSM
    - device attestation (issuance) for my other Yubikeys
- [Caddy Web Server](https://caddyserver.com/docs/automatic-https) as ACME client
    - 24h certificate validity, automatic renewal, JIT issuance
    - mutual TLS with Yubikeys 
    - native integration with Docker and Tailscale

But I won't reinvent the wheel here. Smallstep has detailed tutorials for CA setup of their `step-ca` server and for the Yubikey and reverse proxy mTLS setup.
* [Build a Tiny Certificate Authority For Your Homelab](https://smallstep.com/blog/build-a-tiny-ca-with-raspberry-pi-yubikey/)
* [Access your homelab from anywhere with a YubiKey and mutual TLS](https://smallstep.com/blog/access-your-homelab-anywhere/)

Yes, I did splurge for the USB *true* random number generator. I rationalized it because my CA is running on a Pi 3B instead of a 4 and totally needed the extra entropy horsepower.

I took a look at my threat model and risk tolerance and wanted to make some tweaks to Smallstep's sane defaults and sound guidance. As I said, their default certificate lifetime is 24 hours. That's great for an always-on web service but is untenable for use as a client certificate on a hardware security key. So I extended the lifetime for certificates issued by device attestation to 2 years consulting their [documentation](https://smallstep.com/docs/step-ca/#ssh-certificate-authority).

> **Hey, what happened to those super short lifetimes? Isn't that a risk?**

Kind of. But I'm not that concerned if I lose the Yubikey. The private key can't be used without entering a passcode and can't be extracted from the physical key. Beyond that... **there's no technological defense against the $5 wrench.**

<p align="center">
  <img src="https://imgs.xkcd.com/comics/security.png" alt="Security, by xkcd">
  <br>
  <small>Image source: <a href="https://xkcd.com/538/">xkcd</a></small>
</p>
<br>
All I need to do is add a new hardware key to the CA policies, remove the old one, issue a new cert, and I'm back in business!
<br><br>

And if the Pi shuffles off its silicon coil, I have the setup documented and it takes about an hour and the root certificate should be safe on the HSM - or create a new one. I could make an Ansible playbook for the setup but the time investment doesn't seem worthwhile. See: [Is It Worth the Time?](https://xkcd.com/1205), another obligatory xkcd.

## Future Directions
These are some ways I might extend my CA in the future. Though it might be diminishing returns since I've met these needs in other ways.
- use SSH certificates rather than keys. Combined with a self-hosted SSO solution like [Keycloak](https://www.keycloak.org/) or [Authentik](https://goauthentik.io/). #goals
- put a CA on the public internet. Potential uses:
    - Github Actions certificates rather than hardcoded creds 
    - Testing how to secure a public API and CA
- Practice my Ansible by creating a playbook to bootstrap a new CA
