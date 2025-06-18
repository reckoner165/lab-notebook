## andata

repo: https://github.com/reckoner165/andata

a synth inspired by lyra-8: the idea is to create a web-based drone synth with experimental interactions

I want the scroll-wheel on a mouse to be the primary interaction engine, with various keys on a keyboard dictating what parameter gets changed.
holding several keys down will cause change in all the corresponding parameters in the same direction.
the idea is to bring back weird "multi-touch" interactions that are hard to produce on a computer




#### circa december 2024
- built an early version with faust
- writing synth in faust makes sound reliable, but building UI can be a painful 2-step process

TODO:
- make a simple version of lyra with this! i.e.,
- create 4 tunable oscillators (2 low and 2 high)
- waveform editor to go between sine and square?
- allow groups of two to modulate, and then meta modulate those two groups
- global filter?
- momentary kill-switch for oscillators, adding dynamics
- scroll-wheel + keyboard interaction
Can this be "andata-1"?