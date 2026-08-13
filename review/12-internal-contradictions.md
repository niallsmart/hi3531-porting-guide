# Internal Contradictions to Reconcile

## Audio codec

`doc/13-audio.md` establishes that the TLV320AIC31 is not fitted and that the
NVP1104B codec has no mainline driver. Its final paragraph nevertheless
recommends binding `tlv320aic31xx`.

Please determine whether the intended path is a new NVP1104B codec driver and
update the recommendation accordingly if confirmed.

## MCU watchdog experiment

`doc/20-front-panel-mcu.md`, lines 326–341, records an approximately 60-second
hard-reset timeout. Lines 395–397 later state that the timeout and outcome are
not established.

Please determine whether the latter passage predates the completed experiment.

## MCU watchdog takeover

The MCU document says a replacement userspace daemon must take over watchdog
kicks. Elsewhere it says a reset clears the MCU and a new kernel which never
sends command 7 never arms the watchdog.

Please distinguish a clean boot into a replacement kernel from taking over
UART1 while the running vendor application has already armed the MCU watchdog.

## GIC confidence

`doc/16-porting-roadmap.md`, lines 43–45, calls the GIC bases and SPI numbering
unknown, while `doc/01-soc-overview.md` presents both as established.

Please decide which confidence level is supported by the SDK and live register
evidence and make the two documents consistent.

