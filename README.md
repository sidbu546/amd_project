I have used the gpu droplet to try and run LLaMA 7B. I have followed the following steps:
Step 1: I created gpu droplet then used web console to do my operations:
Step 2: I ran this command to start rocm :docker start rocm
Step 3: I ran this command: docker exec -it rocm /bin/bash to enter the rocm container
Step 4: I ran these commands: python3 -m venv vllm
source vllm/bin/activate to use vllm environment.
Step 5: I ran these commands pip install huggingface_hub
huggingface-cli login. Then I logged into my huggingface account . Accepted the access of the model which was https://huggingface.co/meta-llama/Llama-2-7b 
Step 6: Entered my access token which was in read mode.
Step 7: Then I try to ran python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-2-7b-hf \
  --device cuda \
  --dtype float16 \
  --max-model-len 4096 \
  --gpu-memory-utilization 0.90 this command but this giving me an error
Error: NO NVIDIA cpu detected(More or less this error).
Note: 
Ways to resolve: 
1. I tried using commands like export VLLM_USE_TRITON=0
export VLLM_ATTENTION_BACKEND=TORCH
export VLLM_DEVICE=cuda but still it does not work 
2. I exited from the rocm container 
fill this field export HF_TOKEN with the value and try to run this command too: docker run -d \
  --name llama7b-tgi \
  --restart unless-stopped \
  --cap-add=SYS_PTRACE \
  --security-opt seccomp=unconfined \
  --device=/dev/kfd \
  --device=/dev/dri \
  -e HSA_OVERRIDE_GFX_VERSION=11.0.0 \
  -e HF_TOKEN=$HF_TOKEN \
  -p 8080:80 \
  ghcr.io/huggingface/text-generation-inference:latest-rocm \
  --model-id meta-llama/Llama-2-7b-hf \
  --dtype float16 \
  --max-input-length 4096 \
  --max-total-tokens 4096 
3. I also tried with rocm  MI300x but that is also giving me an error.
