# RF signal analysis & replay (SDR)

A huge amount of everyday hardware talks over sub-GHz radio with no
authentication: remotes, doorbells, weather stations, TPMS, old garage and gate
openers. With a software-defined radio you can watch that spectrum, capture a
transmission, understand how it's encoded, and — on static-code devices — replay
it. The method is the same every time: find the frequency, capture, demodulate,
decode, replay.

All examples are generic. Nothing here is the solution to a specific challenge.

> **Legal first.** *Receiving* and analysing radio is generally fine. *Transmitting*
> — replay, jamming — is strictly regulated almost everywhere and is illegal
> against hardware you don't own. Everything below is for your own devices on a
> test bench, ideally shielded. Know your local rules before you ever key up a
> transmitter.

## 1. The gear

- **RTL-SDR** — cheap, **receive-only**, ~24 MHz–1.7 GHz. Perfect for learning,
  capture and analysis.
- **HackRF / LimeSDR** — **transmit and receive**, wide range. Needed for replay,
  so also where the legal line sits.
- Software: **GQRX** / **SDR++** (live spectrum), **Universal Radio Hacker (URH)**
  (capture + demod + decode), **rtl_433** (auto-decodes hundreds of known
  devices).

## 2. Find the frequency

Most consumer kit lives in the ISM bands: **433.92 MHz** (Europe), **315 MHz**
(US), **868 MHz**, **915 MHz**. Two ways to pin it:

- **FCC ID** — printed on the device; look it up, the filing lists the exact
  operating frequency.
- **Watch the waterfall** — open GQRX around 433.92 MHz, press the remote, and
  look for the spike that appears only while you press.

## 3. Capture the burst

In URH (or `rtl_433 -A` for a quick identify), tune to the frequency, set an
appropriate sample rate, and record while you trigger the device a few times.
You want several clean copies of the same burst to compare.

```
rtl_433 -f 433.92M -A        # identify + show the decoded pulses
```

## 4. Read the modulation

Zoom into the capture. Sub-GHz remotes are almost always one of:

- **OOK / ASK** — the carrier is simply on or off. You'll see clear bursts and
  gaps; decode as pulse widths.
- **FSK** — the frequency shifts between two values for 0 and 1.

URH shows the raw waveform, lets you set the bit length, and converts the bursts
into a bit string. Line up several captures to find the sync/preamble and the
payload.

## 5. Static code vs rolling code

Compare two captures of the *same button*:

- **Identical every time → static code.** The device has no replay protection.
  Captured bits replayed verbatim will trigger it.
- **Different every time → rolling code.** Each press uses a new value from a
  synchronised counter + secret (KeeLoq and friends). A straight replay is
  rejected because that code is now "used".

## 6. Replay (static only, your own device)

With a TX-capable SDR, URH can retransmit the exact samples you captured:

```
# conceptually: load capture -> set same frequency/modulation -> send
```

If it's a static-code device, the gate/bell/plug reacts as if the real remote
was pressed. That's the whole attack — and the whole reason these should not be
static.

## 7. Why rolling codes still aren't magic

Replay fails, but the design has known weaknesses discussed in the literature:
**capture-and-jam** ("rolljam") — jam so the device never hears the first press,
secretly capture it, let the user press again, forward the first code and keep
the second unused for yourself. This needs transmitting (jamming) and is firmly
in illegal-against-others territory; understand it to defend, don't do it in the
wild.

## 8. Defending

- **Don't ship static codes** for anything that controls access. Use rolling
  codes at minimum, authenticated/challenge-response ideally.
- Prefer protocols with real crypto and freshness (nonces/timestamps), not just
  an incrementing counter.
- For high-value entry (cars, gates), move to **bidirectional** challenge-response
  or UWB distance bounding, which defeats simple capture-replay and relay.
- Treat the RF channel like any other input: unauthenticated radio is an open
  door.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
