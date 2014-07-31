navigator.mediaDevices API use cases
====================================

Use cases for MediaDevices API (http://dev.w3.org/2011/webrtc/editor/getusermedia.html#idl-def-MediaDevices) related to handling output device changes.

### Prerequisites

Let's consider a web page with audio player on it, which sound volume can be controlled in some way:
```js
function setCurrentSoundVolume(volume) { /* set sound volume for our player */ }
```

### 1. Change sound volume level depending on current output device

Let's assume we want to remember volume level for each audio output device, and restore it when device becomes active (seems OSX acts in a very similar way). So, I should write some code like this:

```js
function findRegisteredDevice(deviceId) { /* find device in the list of remembered ones */ }
function registerDevice(device) { /* remember device id and current volume */ }
function updateDeviceVolume(deviceId) { /* set new volume for device with given id */ }

// fill in the initial list of audio output devices
navigator.mediaDevices.enumerateDevices(function(list) {
    list.forEach(function(device) {
        if (device.kind === "audiooutput") {
            registerDevice(device);
        }
    });
});

// listen for device changes
navigator.mediaDevices.ondeviceChanged = function() {
    navigator.mediaDevices.enumerateDevices(function(newList) {
        var pluggedOutDevices = getPluggedOutDevices(oldList, newList);
        pluggedOutDevices.forEach(function(device) {
            if (device.kind === "audiooutput") {
                updateDeviceVolume(device.deviceId);
            }
        });

        var pluggedInDevices = getPluggedInDevices(oldList, newList);
        pluggedInDevices.forEach(function(device) {
            if (device.kind === "audiooutput") {
                if (isDeviceActuallyUsedForPlayback(device)) {
                    // let's check if we remember device's last volume
                    registeredDevice = findRegisteredDevice(device.deviceId);
                    if (registeredDevice) {
                        // set last volume level, used for that device
                        setCurrentSoundVolume(registeredDevice.volume);
                    } else {
                        // this is new device, let's register it and remember its volume.
                        registerDevice(device);
                    }
                }
            }
        });
    });
}
```

Here I listen for audio output device changes, and when some of them actually changes I call `enumerateDevices` method and determine devices that were just plugged in or out by analyzing intersections. Devices present in `newList` but not in `oldList` should have been just plugged in, and devices present in `oldList` but not in `newList` were probably plugged out. For plugged out devices, I update information about current sound volume to restore this value when it would become active. For plugged in devices I check if this device is actually used for playback using `isDeviceActuallyUsedForPlayback()`. But seems there is no way to detect, if particular device is actually used for audio playback by underlying OS.

This would be much easier, if we'd have a default device (which is actually playing sound). So the code above would look like this:

```js
navigator.mediaDevices.getDefaultDevice("audiooutput").then(registerDevice);

navigator.mediaDevices.ondefaultdevicechanged = function(newDefaultDevice) {
    if (newDefaultDevice.kind === "audiooutput") {
        var registeredDevice = findRegisteredDevice(newDefaultDevice.deviceId);
        if (registeredDevice) {
            setCurrentSoundVolume(registeredDevice.volume);
        } else {
            registerDevice(device);
        }
    }
}
```

### 2. Mute sound volume when plugging out headphones

Going further let's consider a music player that would play music only when headphones are connected. If headphones are plugged out, it should mute the sound - we don't want to bother people around us with loud noise.

```js
navigator.mediaDevices.ondefaultdevicechanged = function(newDefaultDevice) {
    if (newDefaultDevice.kind === "audiooutput") {
        if (outputDeviceIsLikeHeadphones(newDefaultDevice)) {
            // restore previous sound level for headphones
            tryRestoreSoundVolumeLevelForIt();
        } else {
            // be silent now
            setCurrentSoundVolume(0);
        }
    }
}
```


Here I listen for default audio output device changes and mute sound if changes from headphones to other type. The only question is how to implement `outputDeviceIsLikeHeadphones`? I only see one, very unrelible way to do this using current API - check if device label contains some clue about its type, like this:

```js
function outputDeviceIsLikeHeadphones(device) {
    return device.label.indexOf("phones") != -1;
}
```

It would be much better to have device classes like `"headphones"`, `"speakers"`:

```js
function outputDeviceIsLikeHeadphones(device) {
    return device.deviceClass === "headphones";
}
```
