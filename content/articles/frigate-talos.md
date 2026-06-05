+++
date = '2026-06-15'
title = "Fixing Frigate's RTSP Issues on Kubernetes"
tags = ["Kubernetes"]
+++

# Fixing Frigate's RTSP Issues on Kubernetes

If you've tried deploying [Frigate NVR](https://frigate.video/) on Kubernetes — in this case on Talos Linux — you may have run into a situation where cameras fail to connect and Frigate's API stops working correctly. The root cause comes down to how OpenCV handles RTSP streams by default, and the fix is a single environment variable.

## What's Going On

Frigate's FastAPI layer uses OpenCV under the hood for certain frame-level operations. By default, OpenCV probes RTSP streams over **UDP**. Under most environments this works fine, but in Kubernetes — especially with CNI plugins like Flannel — UDP-based RTSP handling tends to break silently, causing the API to malfunction.

This is the exact issue I hit while deploying Frigate on a homelab Kubernetes cluster running Talos Linux with Flannel as the CNI.

## The Fix

Set the `OPENCV_FFMPEG_CAPTURE_OPTIONS` environment variable in your Frigate pod spec:

```yaml
env:
  - name: OPENCV_FFMPEG_CAPTURE_OPTIONS # Forces RTSP over TCP
    value: "rtsp_transport;tcp"
```

That's it. This tells OpenCV's FFmpeg backend to use TCP transport for RTSP connections, bypassing the UDP handling that breaks in Kubernetes networking environments.

## Closing Notes

This issue likely affects any Kubernetes distribution where the CNI doesn't handle UDP RTSP traffic gracefully — Flannel is a known case, but others may behave similarly. If you're seeing Frigate's API fail or cameras not connecting despite valid RTSP streams outside the cluster, this should be your first thing to check.
