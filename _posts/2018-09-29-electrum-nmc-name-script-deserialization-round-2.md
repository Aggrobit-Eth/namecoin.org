---
layout: post
title: "Electrum-NMC: Name Script Deserialization (Round 2)"
author: Jeremy Rand
tags: [News]
---

I previously wrote about some work on [making Electrum-NMC handle name scripts]({{site.baseurl}}2018/08/06/electrum-nmc-name-script-deserialization.html).  Today I hacked on that code some more.

As you may have noticed from the screenshot in the previous article, I'm initially testing this code with watching-only wallets, since there are fewer moving parts there and I don't have to worry about signing transactions.  When looking for addresses to add to the watching-only wallet, I usually just look through the [Cyphrs block explorer](https://namecoin.cyphrs.com/) and take the first few name transactions that show up of each desired type.  Interestingly, I noticed that a subset of the name transactions I picked from the explorer this time weren't visible in the Electrum-NMC GUI.  Some querying of my ElectrumX server indicated that the transactions were definitely being delivered to Electrum-NMC, and inspecting the wallet file showed that the transactions were definitely being added to the wallet.  But for some reason, Electrum-NMC wasn't displaying them.

After quite a lot of tracing through the code, I figured out that the transactions in question had name scripts that weren't successfully being recognized as name scripts.  Some more tracing led me to figure out that the code was failing to parse any `name_anyupdate` script whose value was the empty string.  Turns out that upstream Electrum has a wildcard-like script matching function that can detect arbitrary data pushes (which I was using) -- but for some reason they don't treat `OP_0` as a data push for the purpose of that wildcard matching.  Pushing the empty string to the stack is implemented by... you guessed it, `OP_0`.  I've filed a GitHub issue with upstream Electrum to inquire whether that's intentional, but in the meantime, I've worked around it in Electrum-NMC's name script parser.

I had previously implemented some special UI code for displaying that a name transaction is a transfer operation.  This is a UX improvement over Namecoin Core, which confusingly displays both incoming and outgoing name transfers identically to name updates.  I had defined a name transfer as "a name transaction for which the wallet owns a name input XOR the wallet owns a name output".  Do you see a problem here?  Yeah, that definition matches **all** `name_new` transactions where the wallet owns the output, because `name_new` transactions *don't have a name input*.  D'oh.  Fixed that.

Up next was displaying the name identifiers in the UI.  Again, I'm trying to improve on Namecoin Core's UX here.  For example, I prefer to make `d/wikileaks` show up as `wikileaks.bit`.  My code also recognizes invalid `d/` names, e.g. names with uppercase characters, and indicates that they're not valid domain names.  Ditto for `id/` names.  In addition, if the namespace is unknown or the identifier isn't valid under the namespace rules, I actually check whether the identifier consists entirely of ASCII printable characters.  If it does, I print it in ASCII; if it doesn't, I print it in hex.  Similar checks are in place for values.  The Namecoin consensus rules have always allowed binary data to exist in identifiers and values, but the UI for this functionality was pretty much always missing.  Electrum-NMC should handle this kind of thing without trouble.  One practical example of how this could be used in the future is storing values as CBOR rather than JSON.  CBOR is substantially more compact, especially when storing binary data like TLS certificates, which means *moar scaling* and *moar savings on transaction fees*.  (CBOR also seems to be a common choice by the DNS community, including people at IETF and ICANN.)

Then, I implemented the `name_show` console command.  It includes SPV verification, just like Electrum's history tab.  Every output JSON field from Namecoin Core is present, although currently none of the optional input fields are supported.  It probably wouldn't be very difficult to hook this into ncdns.  However, it should be noted that Electrum's SPV verification is a weaker security model than ConsensusJ-Namecoin's leveldbtxcache SPV security model.  In addition, there's only one public Namecoin ElectrumX server instance, so the concept of the "longest chain" isn't exactly meaningful.  Even so, it's vastly more secure than centralized inproxies like OpenNIC, and it might be useful for some people.  Hopefully more people will step up to run public Namecoin ElectrumX servers so that the SPV security model can actually work as intended.

Finally, I implemented a first pass at the Manage Names tab.  Display of the name identifier, value, and expiration block count are working.  This was a lot easier than implementing from scratch would be, because the Manage Names tab (referred to in the code as the UNO List widget) is actually just a subclass of the Coins tab (referred to in the code as the UTXO List widget).

So, with that explanation out of the way, here's what you really came here for: *screenshots!*

![A screenshot of name transactions visible in the Electrum-NMC History tab.]({{site.baseurl}}images/screenshots/electrum-nmc/2018-09-29-Names-in-History-Tab.png)

![A screenshot of name data visible in the Electrum-NMC Transaction Details tab.]({{site.baseurl}}images/screenshots/electrum-nmc/2018-09-29-Name-in-Transaction-Details.png)

![A screenshot of a name_show command's output in the Electrum-NMC Console tab.]({{site.baseurl}}images/screenshots/electrum-nmc/2018-09-29-Name-Show-in-Console-Tab.png)

![A screenshot of the Electrum-NMC Manage Names tab.]({{site.baseurl}}images/screenshots/electrum-nmc/2018-10-01-Manage-Names-Tab.png)

This work was funded by NLnet Foundation's Internet Hardening Fund.
