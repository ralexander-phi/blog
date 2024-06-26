---
title: Certificate Authority Trustworthiness
slug: ca-trust
date: 2023-07-18
summary: Challenges facing the CAs and why we still trust them
params:
  guid: 6e7cf359-3482-4a19-82c3-2f986099fdb1
  isPopular: true
  categories:
  - HTTPS
  - Security
---

{{< rawhtml >}}
<style>
div.indent {
  padding-left: 3em;
}
</style>
{{< /rawhtml >}}


The certificate authority (CA) system does an incredible job of solving an impossible challenge.
Think about it.
The CAs measure control of a domain name and then issue TLS certificates that pair cryptographic keys to those names.
They do this on a global scale, often automatically.
It's *impossible* to do this perfectly, and unfortunately, they occasionally fail.

In this post I describe the challenges the CAs face, describe a history of failures, and explain the process we use to maintain confidence in the system in spite of it all.

## Background

The certificate authorities (CAs) solve a foundational key exchange problem for the Internet.
They allow us to authenticate the TLS keys used by web servers, which they do by verifying control of domain names and signing certificates that associate public keys with these names.
Authentication is a critical part of encrypting communications.
Without authentication you may be encrypting with an attacker's key, allowing them to eavesdrop on or tamper with your data in transit.

Methods like certificate pinning work for things like IoT or mobile applications that communicate with a single back-end server.
The developer can hardcode the certificate fingerprint and push an update any time it changes.
But pinning doesn't scale for websites or email.
We need something Internet-scale, and we've got the CAs.


## A history of security concerns

Mozilla maintains a long list of
[CA compliance bugs](https://wiki.mozilla.org/CA/Closed_Incidents)
that tracks over a thousand concerns.
Most of these aren't worth discussing, so let's start with noteworthy CA-related problems from the past year:

{{< rawhtml >}}

<br />
<br />

<div class="indent">
<p>
<b>e-Tugra</b>
<br />
<i>November 2022</i>
<br />
An Internet-facing administration tool used by the e-Tugra CA had a
<a href="https://ian.sh/etugra">wide-open sign-up page</a>
that allowed
<a href="https://ian.sh/">Ian Carrol</a>,
a security researcher, to register an account and view sensitive content.
Of top concern: the confirmation codes used by the email domain control validation (DCV) method were visible.
With this access the security researcher could have started a certificate signing request for a domain they didn't own, chosen the email DCV,
and then intercepted the confirmation code via the administration tool<a href="#foot-1"><sup>1</sup></a>.
e-Tugra fixed the issue when notified, but the community acknowledged
<a href="https://groups.google.com/a/ccadb.org/g/public/c/SXAeHT04TFc/m/69LVodC-HgAJ">"this isn't a little mistake"</a>.
<a href="https://groups.google.com/a/mozilla.org/g/dev-security-policy/c/yqALPG5PC4s/m/ktsJ7LxiAgAJ">Chrome</a>
and
<a href="https://groups.google.com/a/mozilla.org/g/dev-security-policy/c/C-HrP1SEq1A/m/qDXcQu-hBAAJ">Mozilla</a>
distrusted e-Tugra after around six months of discussion.

<p>
<b>Trustcor</b>
<br />
<i>November 2022</i>
<br />
Trustcor appeared to have
<a href="https://groups.google.com/a/mozilla.org/g/dev-security-policy/c/oxX69KFvsm4">"shared corporate officers, operational control and technical integrations"</a>
with Measurement Systems, which "engaged in the distribution of an SDK containing malware".
Chrome distrusted the CA, citing a loss of confidence: "Behavior that attempts to degrade or subvert security and privacy on the web is incompatible with organizations whose CA certificates are included in the Chrome Root Store."
Trustcor contested the claims as "opinion, circumstantial evidence, conjecture, and fear-mongering".

<p>
<b>HiCA</b>
<br />
<i>July 2023</i>
<br />
HiCA used a
<a href="https://github.com/acmesh-official/acme.sh/issues/4659">remote code execution exploit</a>
on users' systems as part of a certificate issuance process.
While HiCA's use of this technique wasn't malicious, the approach was immediately treated as a security vulnerability and
<a href="https://github.com/acmesh-official/acme.sh/commit/327e2fb0a4bdbe4b75339e1cad6d20bda29318d6">fixed</a>
once proper notification was given.
The concern was
<a href="https://groups.google.com/a/mozilla.org/g/dev-security-policy/c/heXVr8o83Ys">briefly discussed</a> on the Mozilla forum,
which acknowledged that HiCA is not a CA.
HiCA was assisting in the certificate issuance process using an actual CA, using a process that was otherwise by the book.
"Literally anyone can do this and do monumentally stupid/insecure things; it's not productive to have a discussion every time this happens."
No action was recommended as the browsers decide on CA trust, not tools or 3rd parties that users choose to assist the issuance process.
HiCA
<a href="https://web.archive.org/web/20230707065028/https://www1.hi.cn/">shut down</a>
soon after these events, citing security incidents.

<p>
<b>Unknown</b>
<br />
<i>March 2022</i>
<br />
An unnamed CA was
<a href="https://symantec-enterprise-blogs.security.com/blogs/threat-intelligence/espionage-asia-governments-cert-authority">hacked</a> 
as part of a suspected state-sponsored hacking campaign also targeting government agencies and defense contractors.
There is "no evidence to suggest [the hackers] were successful in compromising digital certificates".
The lack of clarity on which CA was hacked, how they were hacked, and the lack of adequate public disclosure is troubling.
</div>
<br />
<br />

{{< /rawhtml >}}

There are
[many more](https://www.digicert.com/blog/digicert-statement-trustico-certificate-revocation)
of these that are older, but I think the recent events tell the story well
{{< rawhtml >}}
<a href="#foot-2"><sup>2</sup></a>
{{< /rawhtml >}}
.



The above shows successful attacks, poor security practices, and questionable organizations.
While these are concerning, none of these events appear to have caused actual certificate mis-issuance
{{< rawhtml >}}
<a href="#foot-3"><sup>3</sup></a>
{{< /rawhtml >}}
.
The web browsers carefully consider the risk of maintaining trust relationships when these sorts of events happen, sometimes revoking trust after a thorough review.

Mis-issuances occur somewhat rarely, let's look further back in time.

{{< rawhtml >}}
<br />
<br />
<div class="indent">
<p>
<b>MCS Holding</b>
<br />
<i>March 2015</i>
<br />
MCS Holdings, an intermediary CA,
<a href="https://security.googleblog.com/2015/03/maintaining-digital-certificate-security.html">mis-issued certificates</a>
for various domains, including Google's.
These certificates appeared to be used for an internal man-in-the-middle proxy, but not external to the company.
Google immediately distrusted the intermediary CA and quickly distrusted CNNIC, the root CA used by MCS Holdings.

<p>
<b>ANSSI</b>
<br />
<i>December 2013</i>
<br />
A
<a href="https://security.googleblog.com/2013/12/further-improving-digital-certificate.html">similar incident</a>
occurred with ANSSI.

<p>
<b>DigiNotar</b>
<br />
<i>July 2011</i>
<br />
Hackers
<a href="https://security.googleblog.com/2011/08/update-on-attempted-man-in-middle.html">fraudulently obtained certificates</a>
from the DigiNotar CA for
<code>google.com</code>, <code>cia.gov</code>, <code>mossad.gov.il</code>, <code>microsoft.com</code>,
and more totaling 
<a href="https://www.wired.com/2011/09/diginotar-hacker/">531 certificates</a> in all.
An active man-in-the-middle attack using these certificates was performed against users connecting to Google's services.
The trust in DigiNotar was revoked and the company soon filed for bankruptcy.
</div>
<br />
<br />
{{< /rawhtml >}}

The DigiNotar hack is a textbook example of what we don't want.
The hackers not only compromised the CA, but they also fraudulently issued certificates, established a man-in-the-middle network position and
[intercepted the emails of 300,000 people](https://www.computerworld.com/article/2510951/hackers-spied-on-300-000-iranians-using-fake-google-certificate.html).
Thankfully, the DigiNotar hack is an outlier.

## Trust root inclusions and denials

Each web browser maintains a list of the CAs they trust out-of-the-box.
As we've already seen, this trust can be revoked when problems arise.
But what's the inclusion process?

Review is process heavy, focusing on security assurance and the trustworthiness of the organization operating the CA.
There are independent audits and
[security standards](https://cabforum.org/baseline-requirements-documents/).
In the end, it's a subjective decision with lots of supporting documentation.

For transparency, Mozilla and the 
[CA/Browser Forum](https://cabforum.org/)
use public discussion when deciding if a new CA should be added as a trust root.
There are many boring examples that prompt little debate, like the 
[inclusion request](https://groups.google.com/a/ccadb.org/g/public/c/gk8vbpg5WHo/m/EObfkeUwBQAJ)
for LAWtrust.

The denials can be terrifying:

* [December 2019](https://groups.google.com/g/mozilla.dev.security.policy/c/nnLVNfqgz7g/m/3fnVmRzwBQAJ): Accused of spying, inclusion request was denied and their existing intermediate CA certificate was distrusted.
* [December 2015](https://bugzilla.mozilla.org/show_bug.cgi?id=1232689): "it appears that the owner of this CA has used their certificates to MITM"
* [November 2018](https://groups.google.com/g/mozilla.dev.security.policy/c/fTeHAGGTBqg/m/l51Nt5ijAgAJ): "did not disclose the incident, nor - given that the other two were never revoked - did they apparently perform a scan of their certificates to identify any others."

* [March 2018](https://groups.google.com/g/mozilla.dev.security.policy/c/wCZsVq7AtUY/m/Uj1aMht9BAAJ): "A CA can't simply fix one problem after another as we find them during the inclusion process."

## Attackers focus elsewhere

A rough way to measure the security of a system is to observe how often it is attacked versus adjacent areas.
Attackers have limited resources, so they are biased to choose the weakest links (or perceived weakest).
Here are some examples of how attackers bypass the protections the CAs provide, without attacking the CAs directly:

{{< rawhtml >}}
<ul>
<li>Hackers often have success simply using invalid certificates.
  Users may suffer from
  <a href="https://www.nist.gov/news-events/news/2016/10/security-fatigue-can-cause-computer-users-feel-hopeless-and-act-recklessly">security fatigue</a>,
  and click-through when faced with an active man-in-the-middle attack.
  These attacks are relatively easy to perform and
  <a href="https://www.kali.org/tools/sslsniff/">off-the-shelf</a>
  tools exist.
<li><a href="https://www.phishlabs.com/blog/the-anatomy-of-a-look-alike-domain-attack/">Look-alike domains</a>
  are quite effective.
  An attacker registers a domain that looks like the target domain name.
  Since they own the look-alike domain they can get valid TLS certificates.
  Social engineering is typically used to trick the victim into connecting to the look-alike.
<li>Blocking traffic on port 443 (HTTPS) still works to perform downgrade attacks.
  Savvy users may notice a missing lock icon in the lock icon, but others won't realize there is an issue.
  Tools like HTTPS-only mode and <a href="../hsts-adoption/">HSTS</a> add protection but aren't widely used.
<li>Weak encryption and TLS bugs can be exploited:
  <a href="https://blog.cloudflare.com/logjam-the-latest-tls-vulnerability-explained/">Logjam</a>,
  <a href="https://nvd.nist.gov/vuln/detail/CVE-2016-0701">weak primes</a>,
  <a href="https://www.digicert.com/blog/sweet32-birthday-attack-what-you-need-to-know">Sweet32</a>,
  <a href="https://en.wikipedia.org/wiki/Heartbleed">Heartbleed</a>,
  <a href="https://blog.cloudflare.com/killing-rc4-the-long-goodbye/">RC4</a>,
  <a href="https://en.wikipedia.org/wiki/Lucky_Thirteen_attack">Lucky Thirteen</a>,
  <a href="https://en.wikipedia.org/wiki/POODLE">POODLE</a>,
  <a href="https://thehackernews.com/2015/03/freak-openssl-vulnerability.html">FREAK</a>, and
  <a href="https://blog.cloudflare.com/taming-beast-better-ssl-now-available-across/">BEAST</a>.
<li>Between 2015 and 2020 the Government of Kazakhstan 
  <a href="https://www.zdnet.com/article/kazakhstan-government-is-intercepting-https-traffic-in-its-capital/">repeatedly attempted</a>
  to mandate the installation of a Government-operated root certificate on its citizen's devices.
  This certificate would have allowed the government to perform man-in-the-middle attacks on HTTPS traffic.
  Browser vendors responded by
  <a href="https://blog.mozilla.org/security/2019/08/21/protecting-our-users-in-kazakhstan/">deny-listing these trust roots</a>,
  such that they would not be trusted even if the user manually installed them.
<li>Hackers can easily obtain certificates for sites they've already hacked by manipulating DNS records (ACME DNS-01),
  modifying files on web servers (ACME HTTP-01), or stealing verification codes from email inboxes.
  This works because the CAs validate domain control, not ownership.
  They could even steal existing certificates from servers they've compromised.
  NB: the hackers don't always need to perform a man-in-the-middle attack if they've already compromised an endpoint; they've already carried out a higher-impact attack.
<li>Malware may
  <a href="https://www.fortinet.com/blog/threat-research/deep-analysis-of-driver-based-mitm-malware-itranslator">install its own</a>
  root CA certificates to allow 
  <a href="https://arstechnica.com/gadgets/2021/06/microsoft-digitally-signs-malicious-rootkit-driver/">snooping</a>
  on HTTPS traffic.
</ul>
{{< /rawhtml >}}

Opportunity to attack the CAs exists, but these adjacent attacks are significantly more common.


## Irrevocable Trust

CA root certificates are long lived, up to
[25 years](https://crt.sh/?q=B4585F22E4AC756A4E8612A1361C5D9D031A93FD84FEBB778FA3068B0FC42DC2).
The DigiNotar root certificates that were maliciously used by hackers in 2011 are
[still not expired](https://crt.sh/?q=294f55ef3bd7244c6ff8a68ab797e9186ec27582751a791515e3292e48372d61).
So active revocations are required when issues arise.

Unfortunately, even when the root certificates expire, they don't really expire.
Many consumer devices contain hard-coded trust stores that stop getting updated soon after the initial sale.
This was a concern for Let's Encrypt when the IdenTrust `DST Root CA X3` root certificate expired.
Around
[a third of Android devices](https://letsencrypt.org/2020/11/06/own-two-feet.html)
trusted the expiring root certificate, but hadn't been updated to trust Let's Encrypt's new root.
IdenTrust agreed to sign Let's Encrypt's `ISRG Root X1` certificate *past the expiration* of their own trust root.
This worked because Android 
[doesn't enforce the expiration](https://letsencrypt.org/2020/12/21/extending-android-compatibility.html)
of its trust roots.

While this approach allowed many old Android devices to be usable, it underscores a problem.
The CAs often cannot be distrusted, not via software update nor expiration dates.
As such, some devices will forever trust the problematic CAs referenced earlier in the post.


## Government control of CAs

A
[recurring topic of discussion](https://groups.google.com/g/mozilla.dev.security.policy/c/cJSNqUZ4Dsc/m/plq98_wFu3MJ)
is government coercion of the CAs.
Every CA operates within the jurisdiction of a government which can exert legal pressure on the CA.
Many countries have laws that compel technology companies to assist the government in certain circumstances.
The CAs promise not to mis-issue certificates, but a request from their government could supersede that promise.

Understanding the legal exposure of a CA is a complicated question of foreign law.
Since revoking a trust root isn't always possible, it's also an exercise in predicting how laws may change.

Given those concerns, you may be surprised to know that some of the CAs are directly operated by government agencies.
Here are several from the
[Microsoft trust store](https://ccadb.my.salesforce-sites.com/microsoft/IncludedCACertificateReportForMSFT):

{{< rawhtml >}}
<ul>
<li>Department of Defence Australia
  <a href="https://crt.sh/?q=209E956AF04DF3996507C887D356230D6EB49FDBDD2D8A058FF50B8F80F690AA">(cert)</a>
<li>Government of Brazil, Instituto Nacional de Tecnologia da Informação (ITI) 
  <a href="https://crt.sh/?q=6E0BFF069A26994C15DE2C4888CC54AF84882E5495B7FBF66BE9CCFFEC7489F6">(cert)</a>
<li>Government of Finland, Population Register Centre’s (Väestörekisterikeskus, VRK) 
  <a href="https://crt.sh/?q=5546A52504FBA74F61FFD4890067529ADE3B9C9D07E502592831CCDA9B369FD3">(cert)</a>
<li>Government of Hong Kong (SAR), Hongkong Post, Certizen 
  <a href="https://crt.sh/?q=3945E08A8D4A0554B7605A7B355B10188E3EF842C76A805C54E3657C4D041AAA">(cert)</a>
<li>Government of India, Ministry of Communications & Information Technology, Controller of Certifying Authorities (CCA) 
  <a href="https://crt.sh/?q=9A3FD3176798E842DDCB12C262F11CFACCA70A8B84C6EA6FDA30842A95A94CD8">(cert)</a>
<li>Government of Korea, KLID 
  <a href="https://crt.sh/?q=407C276BEAD2E4AF0661EF6697341DEC0A1F9434E4EAFB2D3D32A90549D9DE4A">(cert)</a>
<li>Government of Lithuania, Registru Centras 
  <a href="https://crt.sh/?q=7707BB2BE9F7CE057060B8308C3BC087B56529B3638EAF5B2A8049C8E15ED720">(cert)</a>
<li>Government of Portugal, Sistema de Certificação Electrónica do Estado (SCEE) / Electronic Certification System of the State
  <a href="https://crt.sh/?q=488E134F30C5DB56B76473E608086842BF21AF8AB3CD7AC67EBDF125D531834E">(cert)</a>
<li>Government of Saudi Arabia, NCDC
  <a href="https://crt.sh/?q=34BB34E14FAED0D3392F2FC441C0ECD5FD88AD88118DF2D1BA76CDEC1EEA10B8">(cert)</a>
<li>Government of South Africa, Post Office Trust Centre
  <a href="https://crt.sh/?q=FD7CCA48B745213470F503EC6E969C447FC12D4DD5AA2A7CC6C3D5ABBD962B33">(cert)</a>
<li>Government of Spain, Autoritat de Certificació de la Comunitat Valenciana (ACCV) 
  <a href="https://crt.sh/?q=9A6EC012E1A7DA9DBE34194D478AD7C0DB1822FB071DF12981496ED104384113">(cert)</a>
<li>Government of Spain, Dirección General de la Policía – Ministerio del Interior – España 
  <a href="https://crt.sh/?q=739710C5245E33EC8A243A1B20048FC9D5F4528599213845C164D004B8B667F9">(cert)</a>
<li>Government of Spain, Fábrica Nacional de Moneda y Timbre (FNMT) 
  <a href="https://crt.sh/?q=EBC5570C29018C4D67B1AA127BAF12F703B4611EBC17B7DAB5573894179B93FA">(cert)</a>
<li>Government of Sweden (Försäkringskassan) 
  <a href="https://crt.sh/?q=8F9ADB6D895DAB5ADF5C3D3FAB83927BE0FB64EF82485C62280D584E8BD55D22">(cert)</a>
<li>Government of Taiwan, Government Root Certification Authority (GRCA) 
  <a href="https://crt.sh/?q=70B922BFDA0E3F4A342E4EE22D579AE598D071CC5EC9C30F123680340388AEA5">(cert)</a>
<li>Government of The Netherlands, PKIoverheid (Logius) 
  <a href="https://crt.sh/?q=3C4FB0B95AB8B30032F432B86F535FE172C185D0FD39865837CF36187FA6F428">(cert)</a>
<li>Government of Turkey, Kamu Sertifikasyon Merkezi (Kamu SM) 
  <a href="https://crt.sh/?q=46EDC3689046D53A453FB3104AB80DCAEC658B2660EA1629DD7E867990648716">(cert)</a>
<li>Government of Uruguay, Agency for E-Government and Information Society (AGESIC) 
  <a href="https://crt.sh/?q=5533A0401F612C688EBCE5BF53F2EC14A734EB178BFAE00E50E85DAE6723078A">(cert)</a>
<li>Korea Information Security Agency (KISA) 
  <a href="https://crt.sh/?q=6FDB3F76C8B801A75338D8A50A7C02879F6198B57E594D318D3832900FEDCD79">(cert)</a>
<li>Macao Post and Telecommunications Bureau 
  <a href="https://crt.sh/?q=0E30D04A94DCC423E26FDC4C24AFCF4923BD80D83B661B06A2F5856361560407">(cert)</a>
<li>Swiss BIT, Swiss Federal Office of Information Technology, Systems and Telecommunication (FOITT) 
  <a href="https://crt.sh/?q=6EC6614E9A8EFD47D6318FFDFD0BF65B493A141F77C38D0B319BE1BBBC053DD2">(cert)</a>
<li>Thailand National Root Certificate Authority (Electronic Transactions Development Agency) 
  <a href="https://crt.sh/?q=2A8DA2F8D23E0CD3B5871ECFB0F42276CA73230667F474EEDE71C5EE32CC3EC6">(cert)</a>
</ul>
{{< /rawhtml >}}

Historic trust also existed for:

{{< rawhtml >}}
<ul>
<li>China Internet Network Information Center (CNNIC) (discussed earlier)
<li>Government of France (ANSSI, DCSSI)  (discussed earlier)
<li>Government of Japan, Ministry of Internal Affairs and Communications 
<li>Government of Latvia, Latvian State Radio & Television Centre (LVRTC) 
<li>Government of Mexico, Autoridad Certificadora Raiz de la Secretaria de Economia 
<li>Government of Venezuela, Superintendencia de Servicios de Certificación Electrónica (SUSCERTE) 
<li>Post of Slovenia 
<li>The Uruguayan Post, “El Correo Uruguayo” 
<li>U.S. Federal Public Key Infrastructure (US FPKI) <a href="https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/microsoft-trusted-root-store-removal-of-us-federal-common-policy">(removal)</a>
</ul>
{{< /rawhtml >}}

The
[Mozilla list](https://ccadb.my.salesforce-sites.com/mozilla/IncludedCACertificateReport)
and
[Google list](https://chromium.googlesource.com/chromium/src/+/main/net/data/ssl/chrome_root_store/root_store.md)
are much shorter than Microsoft's list.
Unfortunately, Microsoft doesn't operate a public discussion forum, so the purpose and justification of these inclusions is not apparent.

One of the most important functions of a government is to provide services to its people.
It's common for governments to provide security services, issue identity documents, and handle delivery of postal mail, for example.
I don't think it's unusual for governments to seek to provide Internet-based identity services.
But there is a conflict of interest, as governments also perform law enforcement, intelligence, and military operations.
The global reach a government-operated CA has may not be appropriate.

## Trusting the system

With all these problems, why do we still trust the CA system?

One reason is the lack of a better alternative.
I'm planning to post about
[DNSSEC+DANE](https://caniuse.com/dnssec)
soon, but in short: it's a mess.
This is an incredibly hard problem space, and nothing else is viable.

The web browsers have done an excellent job defining security standards, reviewing inclusion requests, considering revocations, and being transparent to the users about their decisions.
It's a dynamic process needing constant attention and vigilance.
Requiring certificate transparency logs (a public log of every issued certificate) provides great tooling to audit the issuance practices and 
[swiftly detect problems](https://sslmate.com/certspotter/).
Recent improvements like
[Certificate Authority Authorization (CAA)](https://letsencrypt.org/docs/caa/)
reduce the impact of certain classes of CA security incidents.
Attacks like the DigiNotar hack are much easier to detect these days and we have tools to reduce impact.


## Call to action

The large number of attacks impacting adjacent systems is a strong signal that effort is best spent securing those adjacent systems.
Security isn't about perfection, it's about strengthening weak areas.
With that said, I think there's still high-value work to be done here.

### Transparent inclusion decisions

This post is heavily references actions of the Mozilla trust store, largely due to the public discussion forum they use.
Without such public discussion users have no idea what the inclusion process looked like.
Sometimes 
[objections are raised](https://bugzilla.mozilla.org/show_bug.cgi?id=1204656#c24)
and the justification used for inclusion helps concerned users understand the nuance of the role the CAs fill.
Microsoft, Apple, and Google own major web browsers but do not provide an equivalent open discussion forum.
Each browser trusts a distinct set of trusted CAs, so the justification for inclusion can't always be inferred from Mozilla's forum.

You can review these lists here:

* [Chrome](https://chromium.googlesource.com/chromium/src/+/main/net/data/ssl/chrome_root_store/root_store.md)
* [Apple](https://support.apple.com/en-us/HT213824)
* [Microsoft](https://ccadb.my.salesforce-sites.com/microsoft/IncludedCACertificateReportForMSFT)
* [Mozilla](https://ccadb.my.salesforce-sites.com/mozilla/IncludedCACertificateReport)


### Bug Bounties

None of the CAs offer bug bounties, they instead rely on private auditors.
There's *clearly* low-hanging fruit the auditors are missing.
Bug bounties are a great way to encourage altruistic hackers to take a look at your security while providing
[safe harbor](https://www.microsoft.com/en-us/msrc/bounty-safe-harbor).
Without these incentives, less scrupulous hackers will look anyway and may sell their findings to malicious actors.

### Upgradable trust stores

Some devices can be challenging to update, especially if they are abandoned by the manufacturer.
Let's Encrypt had this issue when they were getting started as a CA, particularly for Android devices.
The only way to mass deploy updates to the trust store on Android is via over-the-air (OTA) updates.
Thankfully Android is fixing this issue by adding
[updatable trust stores](https://www.xda-developers.com/android-14-root-certificates-updatable/).
All devices should support updating trust stores.

### Smaller trust stores

If Chrome doesn't trust a CA, why should my Firefox browser trust it?
And vice versa.
Any competently operated website wouldn't use a CA that isn't widely trusted, so the loss of functionality should be marginal.
When a distrust action is taken, it's common for all the trust stores to agree on revocation, but decisions don't always happen at the same time.
Custom browser builds with a security focus should use a trust store that only includes trust roots that are in all the major browser trust stores.
Better UX can be helpful for
[end-user pruning](https://superuser.com/questions/1379589/remove-all-certificate-authorities-from-a-firefox-profile)
of trust roots.

## Footnotes

{{< rawhtml >}}
<p id="foot-1">
1. CAA could protect a domain from this sort of attack.
Imagine an attacker who can spoof challenge responses for a single CA, thereby fraudulently creating certificates.
A domain owner who had enabled CAA for their domain would be immune to this attack unless they had chosen the affected CA.

<p id="foot-2">
2. I've excluded a 
<a href="https://techmonitor.ai/technology/cybersecurity/entrust-ransomware-attack">security incident</a>
at Entrust as it
<a href="https://groups.google.com/a/mozilla.org/g/dev-security-policy/c/gPYHPhuxHJc/m/yucicLhPAwAJ">didn't impact</a>
their PKI business.

<p id="foot-3">
3. This post is focused on certificates used to secure HTTPS/TLS connections but there are other uses, like code signing.
Incidents like
<a href="https://arstechnica.com/gadgets/2021/06/microsoft-digitally-signs-malicious-rootkit-driver/">this 2021 security lapse</a>
where an attacker tricked Microsoft into signing malware are out of scope.

<p id="foot-4">
4. I've 
<a href="../name-non-constraint/">posted before</a>
about the issues with name constraints.
Even when there's intent to constrain a CA's root certificate to a particular TLD, like discussed in
<a href="https://groups.google.com/g/mozilla.dev.security.policy/c/vjXyml8Hy-E/m/5JUs8e3YDAAJ">this request</a>,
the name constraint feature isn't used.
{{< /rawhtml >}}

