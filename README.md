Welcome. This is my personal fork of the waveshare s2 audio config provided by sw3Dan. There are some minor changes that have worked as improvements for me, but the base code is largely unchanged. Here are the fixes I've added:

- Added more functionality to the LED section. Fixed the pulse setting, where without a minimum and maximum it would only show a solid color due to min/max brightness. A personal preference, but added different colors for each phase of the voice assistant response (using swirl effects because... well I thought it looked pretty). Mostly as an indicator of when your voice is being picked up and cut off by the VAD as well as showing how long home assistant was taking to answer.
- While there is a mic gain setting in the main, adjusting gain_factor for va helped with comprehension in home assistant. At default (1) audio files were inaudible, and even though whisper (I'm using wyoming-small-en-int-8) could process them, adjusting and using debugging in home assistant let me get a better signal to noise ratio for picking up requests further away.
- Added a slider in home assistant to adjust the va gain_factor, as well as noise_suppression_level. Did not add similar settings to the microwakeword, as it appears to be functioning without modifications, but I might experiment and commit a change. 

sw3Dan's info from the main included below:

ESPHome configuration for enabeling WAVESHARE-S3-AUDIO-BOARD (https://www.waveshare.com/esp32-s3-audio-board.htm)
to be used as a HomeAssistant Voice Satellite. Features such as simultanious music/and announcements and
continious on-board wake-word detection.

The device's DAC and ADC share pins for the i2s bus so to be able to configure two i2s_audio points for 
simultanious audio in/out I had to path the DAC (es8311) component, adding a setting to force it to become 
i2s master while still configuring the ESP and ADC to act as i2s slaves. I also had to fix MCLK/BLCK calculations.

example code features:
* complete Voice Assistant setup
* Onboard Wake-Word detection
* Led animations and event sounds
* working control buttons
* exposed announcement and music media_players
* built in alarm and timer

Steps to use the custom ES8311 component:
``` yaml
substitutions:
  i2c_id: internal_i2c
  i2s_mclk_multiple: 256
  i2s_bps_spk: 16bit
  i2s_bps_mic: 16bit
  i2s_sample_rate_spk: 16000
  i2s_sample_rate_mic: 16000 # must be 16000 for voice_assistant to work(?)
  i2s_use_apll: true
  
external_components:
  - source:
      type: git
      url: https://github.com/sw3Dan/waveshare-s2-audio_esphome_voice
      ref: main
    components: [ es8311 ]
    refresh: 0s

i2c:
  - id: $i2c_id
    sda: GPIO11
    scl: GPIO10
    scan: true

i2s_audio:
  - id: i2s_output
    i2s_lrclk_pin: 
      number: GPIO14
      allow_other_uses: true
    i2s_bclk_pin:  
      number: GPIO13
      allow_other_uses: true
    i2s_mclk_pin: # <-- for ESP to know what port to configure MCLK output
      number: GPIO12

  - id: i2s_input
    i2s_lrclk_pin:  
      number: GPIO14
      allow_other_uses: true
    i2s_bclk_pin:  
      number: GPIO13
      allow_other_uses: true

audio_dac:
  - platform: es8311
    id: es8311_dac
    i2c_id: $i2c_id
    use_mclk: true
    sample_rate: $i2s_sample_rate
    bits_per_sample: $i2s_bps_spk
    force_master: true # <-- New: to configure device as i2s master
    mclk_multiple: i2s_mclk_multiple # <-- New: for the i2s bus MCLK/BLCK calculations

audio_adc:
  - platform: es7210
    id: adc_mic
    i2c_id: $i2c_id
    sample_rate: $i2s_sample_rate
    bits_per_sample: $i2s_bps_mic

microphone:
  - platform: i2s_audio
    id: i2s_mics
    i2s_din_pin: GPIO15
    adc_type: external
    pdm: false
    i2s_audio_id: i2s_input
    i2s_mode: secondary # <-- so that ESP wont generate LRCLK/BLCK
    mclk_multiple: $i2s_mclk_multiple
    sample_rate: $i2s_sample_rate_mic # must be 16000 for VA (?)
    bits_per_sample: $i2s_bps_mic
    
speaker:
  - platform: i2s_audio
    id: i2s_audio_speaker
    i2s_dout_pin: GPIO16
    i2s_audio_id: i2s_output
    i2s_mode: secondary # component patch will force ES8311 to take command
    dac_type: external
    timeout: never
    buffer_duration: 100ms
    audio_dac: es8311_dac
    sample_rate: $i2s_sample_rate_spk
    bits_per_sample: $i2s_bps_spk
    use_apll: $i2s_use_apll # dont know if this is enforced when in 'secondary' i2s mode
    mclk_multiple: $i2s_mclk_multiple
    channel: stereo

  - platform: mixer
    id: mixing_speaker
    output_speaker: i2s_audio_speaker
    num_channels: 2
    source_speakers:

** Then create speaker: speaker/resamplers as needed ** 
```

If you need a camera include this yaml package.
This package comes with a preconfigured camera and a bunch of camera control is exposed to HomeAssistant.
``` yaml
packages:
  remote_package_shorthand: github://esphome/sw3Dan/waveshare-s2-audio_esphome_voice/waveshare_camera_pkg.yaml@main
```

General TODO/wishlist for the device.
* filter speaker output from mic input using IDF-SR library (IN PROGRESS!)
* UI: disable LEDS
* UI: disable Voice/wake-word
* UI/ESP: Forward output to external speaker (music assistant announcement support?)
* ESP: lower volume on room speakers when wake-word detected
* UI: expose mic and model sensitivity calibration settings
* UI: select buttons behavior
* ESP: long-press (volume) and double-click  (deactivate Voice) button support
* ESP: Volume percentage Led animation
* ESP: rotary rainbow - assistant working
* ESP: alarm flash led animation
* ESP: pulsing red - no connection led animation
* ESP: code cleanup - change code structure and split into packages
* UI: structure/names
* ESP: mmWave module (control mic activation to limit room exposure)
* ESP: stream/listen to audio (mic output volume trigger)
* ESP: fix RTC
* UI: set TimeZone
