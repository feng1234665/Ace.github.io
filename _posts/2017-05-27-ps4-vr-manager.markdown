---
layout: post
title: ps4 VRManager
date: 2017-05-27
---

能上代码不BB
```
using UnityEngine;
using UnityEngine.VR;
using System.Collections;
#if UNITY_PS4
using UnityEngine.PS4.VR;
using UnityEngine.PS4;
#endif

public class VRManager : MonoBehaviour
{
    public float renderScale = 1.4f; // 1.4 is Sony's recommended scale for PlayStation VR
    public bool showHmdViewOnMonitor = true; // Set this to 'false' to use the monitor/display as the Social Screen
    private static VRManager _instance;

    public static VRManager instance
    {
        get
        {
            if (_instance == null)
            {
                _instance = FindObjectOfType<VRManager>();
                DontDestroyOnLoad(_instance.gameObject);
            }

            return _instance;
        }
    }

    void Awake()
    {
        if (_instance == null)
        {
            _instance = this;
            DontDestroyOnLoad(this);
        }
        else if (this != _instance)
        {
            // There can be only one!
            Destroy(gameObject);
        }
    }

    public void BeginVRSetup()
    {
        StartCoroutine(SetupVR());
    }

    IEnumerator SetupVR()
    {
#if UNITY_PS4
        // Register the callbacks needed to detect resetting the HMD
		Utility.onSystemServiceEvent += OnSystemServiceEvent;
        PlayStationVR.onDeviceEvent += onDeviceEvent;

#endif

        VRSettings.LoadDeviceByName("PlayStationVR");

        // WORKAROUND: At the moment the device is created at the end of the frame so
        // changing almost any VR settings needs to be delayed until the next frame
        yield return null;
        VRSettings.enabled = true;
        VRSettings.renderScale = renderScale;
        VRSettings.showDeviceView = showHmdViewOnMonitor;
    }

    public void BeginShutdownVR()
    {
        StartCoroutine(ShutdownVR());
    }

    IEnumerator ShutdownVR()
    {
        VRSettings.LoadDeviceByName("None");
        yield return null;

        VRSettings.enabled = false;
        VRSettings.showDeviceView = false;

#if UNITY_PS4
        // Unregister the callbacks needed to detect resetting the HMD
        Utility.onSystemServiceEvent -= OnSystemServiceEvent;
        PlayStationVR.onDeviceEvent -= onDeviceEvent;
        PlayStationVR.SetOutputModeHMD(false, 90);
#endif
        Camera.main.fieldOfView=60;
        Camera.main.ResetAspect();

        // WARNING: This is specific to the sample and essentially restarts the
        // game. Please remove this for your own title and handle shutdown gracefully
        UnityEngine.SceneManagement.SceneManager.LoadScene(0);
    }

    public void SetupHMDDevice()
    {
#if UNITY_PS4
        // The HMD Setup Dialog is not displayed on the social screen in separate
        // mode, so we'll force it to mirror-mode first
        VRSettings.showDeviceView = false;

        // Show the HMD Setup Dialog, and specify the callback for when it's finished
        HmdSetupDialog.OpenAsync(0, OnHmdSetupDialogCompleted);
#endif
    }

    public void ToggleHMDViewOnMonitor(bool showOnMonitor)
    {
        showHmdViewOnMonitor = showOnMonitor;
        VRSettings.showDeviceView = showHmdViewOnMonitor;
    }

    public void ToggleHMDViewOnMonitor()
    {
        showHmdViewOnMonitor = !showHmdViewOnMonitor;
        VRSettings.showDeviceView = showHmdViewOnMonitor;
    }

    public void ChangeRenderScale(float scale)
    {
        VRSettings.renderScale = scale;
    }

#if UNITY_PS4
    // HMD recenter happens in this event
    void OnSystemServiceEvent(UnityEngine.PS4.Utility.sceSystemServiceEventType eventType)
    {
        Debug.LogFormat("OnSystemServiceEvent: {0}", eventType);

        switch (eventType)
        {
            case Utility.sceSystemServiceEventType.RESET_VR_POSITION:
                InputTracking.Recenter();
                break;
        }
    }
#endif

#if UNITY_PS4
    // Detect completion of the HMD dialog and either proceed to setup VR, or throw a warning
    void OnHmdSetupDialogCompleted(DialogStatus status, DialogResult result)
    {
        Debug.LogFormat("OnHmdSetupDialogCompleted: {0}, {1}", status, result);

        switch (result)
        {
            case DialogResult.OK:
                StartCoroutine(SetupVR());
                break;
            case DialogResult.UserCanceled:
                Debug.LogWarning("User Cancelled HMD Setup!");
                BeginShutdownVR();
                break;
        }
    }
#endif

#if UNITY_PS4
    // This handles disabling VR in the event that the HMD has been disconnected
    bool onDeviceEvent(PlayStationVR.deviceEventType eventType, int value)
    {
        Debug.LogFormat("### onDeviceEvent: {0}, {1}", eventType, value);
        bool handledEvent = false;

        switch (eventType)
        {
            case PlayStationVR.deviceEventType.deviceStopped:
                BeginShutdownVR();
                handledEvent = true;
                break;
            case PlayStationVR.deviceEventType.StatusChanged:   // e.g. HMD unplugged
                VRDeviceStatus devstatus = (VRDeviceStatus)value;
                Debug.LogFormat("DeviceStatus: {0}", devstatus);
                if (devstatus != VRDeviceStatus.Ready)
                {
                    // TRC R4026 suggests showing the HMD Setup Dialog if the device status becomes non-ready
                    if (VRSettings.loadedDeviceName == "None")
                        SetupHMDDevice();
                    else
                        BeginShutdownVR();
                }
                handledEvent = true;
                break;
            case PlayStationVR.deviceEventType.MountChanged:
                VRHmdMountStatus status = (VRHmdMountStatus)value;
                Debug.LogFormat("VRHmdMountStatus: {0}", status);
                handledEvent = true;
                break;
        }

        return handledEvent;
    }
#endif
}



```