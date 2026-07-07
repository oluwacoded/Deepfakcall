# Running Teddy MFG Call App on a GPU host (smooth, real-time swaps)

## Why this is needed
- **Replit has no GPU**, so the celebrity ("Load Model" / DFM) swap runs on CPU at
  ~370 ms/frame — that's the lag.
- **The code is already GPU-ready.** At startup `web_pipeline._pick_device()`
  detects an NVIDIA GPU (CUDA) and uses it automatically — **no code change**.
  On a GPU the DFM swap drops to roughly **10–30 ms/frame**, i.e. smooth,
  expression-tracking swaps like the TikTok clips.
- So "move to a GPU host" = rent a GPU server, put this code + the model files on
  it, install the GPU build of onnxruntime, and run the same `python web_server.py`.

## What it costs (please read)
- A suitable GPU (RTX 3090 / 4090 / A10 / L4) rents for about **$0.30–$1.00/hour**.
- **Always-on 24/7 ≈ $220–$720/month.** If you only switch it on for calls you pay
  per hour, but then you must start/stop the server yourself.
- The GPU host bills you directly (their account, not Replit).

## What you need to move
1. **Code** — GitHub: `https://github.com/oluwacoded/Deepfakcall`
2. **Model files (~1.4 GB, NOT in GitHub — must be copied separately):**
   - `userdata/models/*.dfm`  (the celebrity models, ~718 MB each)
   - `modelhub/onnx/**`        (the small face-detector models)
3. **A GPU host** with an NVIDIA GPU, CUDA 12 and cuDNN. Easiest is a host image
   that already includes CUDA/cuDNN so you don't install drivers.

## Recommended easy hosts
- **RunPod** (runpod.io) — pick a "PyTorch 2.x / CUDA 12" pod, expose HTTP port
  5000. ~\$0.34/hr for an RTX 4090. Good balance of easy + cheap.
- **Lambda Cloud** (lambdalabs.com) — on-demand GPU VM, CUDA preinstalled.
- **Any Ubuntu 22.04 VM** with an NVIDIA GPU where `nvidia-smi` works.

## Steps (generic Ubuntu 22.04 + CUDA 12 GPU box)
```bash
# 1. SSH in, confirm the GPU is visible:
nvidia-smi                     # must list your GPU

# 2. Get the code:
git clone https://github.com/oluwacoded/Deepfakcall.git app && cd app

# 3. Install the GPU deps:
python3 -m venv venv && source venv/bin/activate
pip install -r requirements-gpu.txt

# 4. Copy the models up (they are NOT in GitHub). From your Replit shell,
#    replace user@HOST with the GPU box, then run:
#      rsync -avP userdata/models/*.dfm  user@HOST:~/app/userdata/models/
#      rsync -avP modelhub/onnx/         user@HOST:~/app/modelhub/onnx/
#    (Or upload them to any cloud storage and download them on the box.)
#    The layout on the GPU box must match this repo, e.g.
#    userdata/models/Amber_Song.dfm

# 5. Set the session secret:
export SESSION_SECRET="<a long random string>"

# 6. Open the port: allow inbound TCP 5000 (RunPod: expose port 5000).

# 7. Run it:
python web_server.py
```

## Verify the GPU is actually being used
In the startup logs you want to see:
```
[Pipeline] Using GPU device: ...
```
If instead you see `[Pipeline] No GPU found; using CPU inference.`, CUDA/cuDNN is
not visible to onnxruntime — see Troubleshooting.

Then open `http://<host-ip>:5000` (or the RunPod-provided URL), load a model,
start the camera. The swap should now be smooth.

## Production niceties (optional)
- Put **nginx + HTTPS** in front — phones block camera/mic on plain `http`
  (only `localhost` is exempt), so remote users need `https`.
- Use **systemd** (or pm2) to keep the server running and restart on crash.
- For the video-call feature across strict/mobile networks, add a **TURN server**
  (e.g. coturn) so WebRTC can connect through NATs.

## Troubleshooting
- **"No GPU found" in logs:** CUDA/cuDNN not visible to onnxruntime. Use a CUDA 12
  host image, confirm `nvidia-smi` works, and make sure you installed
  `requirements-gpu.txt` (onnxruntime-gpu), not the CPU `requirements.txt`. If the
  host only has CUDA 11.8, pin `onnxruntime-gpu==1.16.3`.
- **Camera won't start on a phone:** you need HTTPS (see nginx note above).
- **Model dropdown is empty:** the `.dfm` files didn't get copied to
  `userdata/models/` — recheck step 4 and the directory layout.
