# SD3.5 ControlNet Pipeline

This repository provides a script for running Stable Diffusion 3.5 with multiple ControlNets (Canny, Depth, Blur) using the Hugging Face `diffusers` library. This is an attempt at using the SD3.5 ControlNets and includes custom workarounds to address specific requirements that may not yet be handled by the standard library.

## Custom Implementation Details

This implementation uses a custom pipeline class `SD3ControlNetPipelineWithCannyFix` which inherits from `diffusers.StableDiffusion3ControlNetPipeline` to enable correct functionality with SD3.5 ControlNets.

### Canny ControlNet Preprocessing

The standard `diffusers` pipeline does not apply the special preprocessing required for the SD3.5 Canny ControlNet. The custom pipeline overrides the image preparation step to apply the correct transformation to the Canny control image latents. This is crucial for getting correct results from the Canny ControlNet.

### Multi-ControlNet Order

The script is designed to use multiple ControlNets simultaneously (Canny, Depth, and Blur). It uses `diffusers.SD3MultiControlNetModel` and assumes a specific order for the control images and conditioning scales:

1.  Canny
2.  Depth
3.  Blur

When providing control images or setting parameters like `controlnet_conditioning_scale`, they must be in this order.

## Model Preparation

The ControlNet models for SD3.5 are often distributed as single `.safetensors` files. These must be converted into the `diffusers` format before they can be used with this script.

You can use the conversion script provided in the `diffusers` library. Here is an example command:

```bash
python <path_to_diffusers>/scripts/convert_sd3_controlnet_to_diffusers.py \
    --checkpoint_path "/path/to/your/sd3.5_large_controlnet_depth.safetensors" \
    --output_path "/path/to/your/sd3.5_large_controlnet_depth_diffusers"
```

Repeat this process for each ControlNet model (Canny, Depth, Blur).

## Usage

### Interactive Mode (no arguments)
```bash
python sd3.5_diffusers_control.py
```
Runs in interactive mode where you can:
- Generate images using default configuration
- Create/modify `config.json` file with your settings
- Press ENTER to regenerate with new settings
- Press Ctrl+C to exit

The script will monitor `config.json` and reload it for each generation, allowing you to experiment with different settings without restarting the script.

### Batch Configuration Mode
```bash
python sd3.5_diffusers_control.py --config batch_config.json
```
Processes multiple configurations from a JSON file. The JSON file must contain an array of configuration objects.

## JSON Configuration Format

The configuration file must be a JSON array containing one or more configuration objects:

```json
[
    {
        "prompt": "anime style artwork",
        "input_image": "./inputs/image1.png",
        "output_dir": "outputs/batch1",
        "final_output": "outputs/batch1/anime.png",
        "seed": 42
    },
    {
        "prompt": "studio ghibli style",
        "input_image": "./inputs/image2.png",
        "output_dir": "outputs/batch2",
        "final_output": "outputs/batch2/ghibli.png",
        "seed": 123
    }
]
```

## Environment Variables

The following environment variables can be set to control the pipeline behavior:

- `LOCAL_FILES_ONLY`: Set to "true" (default) to only use locally cached models, "false" to allow downloading
- `DEPTH_MODEL_TYPE`: Choose depth estimation model - "dpt" or "depth_anything_v2" (default)
- `DEPTH_ANYTHING_MODEL`: Specify Depth Anything model when using depth_anything_v2 (default: "depth-anything/Depth-Anything-V2-Base-hf")
- `LOAD_EACH_MODEL`: Set to "true" (default) to load models individually, "false" to load all at once

Example:
```bash
export DEPTH_MODEL_TYPE="depth_anything_v2"
export DEPTH_ANYTHING_MODEL="depth-anything/Depth-Anything-V2-Large-hf"
export LOCAL_FILES_ONLY="false"
python sd3.5_diffusers_control.py
```

## Available Configuration Parameters

All parameters are optional. If not specified, the default value from the Config class will be used.

### Model Paths
- `model_repo`: HuggingFace model repository (default: "stabilityai/stable-diffusion-3.5-large")
- `depth_controlnet_path`: Path to depth controlnet model (diffusers format)
- `canny_controlnet_path`: Path to canny controlnet model (diffusers format)
- `blur_controlnet_path`: Path to blur controlnet model (diffusers format)
- `depth_model`: Depth estimation model (default: "Intel/dpt-hybrid-midas")

### Input/Output
- `input_image`: Path to input image
- `output_dir`: Output directory
- `depth_output`: Path for depth control image
- `canny_output`: Path for canny control image
- `blur_output`: Path for blur control image
- `final_output`: Path for final generated image

### Generation Parameters
- `prompt`: Text prompt for generation
- `negative_prompt`: Negative prompt
- `aspect_ratio`: Aspect ratio mode - "auto" (default), "square", "landscape", or "portrait"
  - "auto": Automatically detect from input image
  - "square": 1024x1024
  - "landscape": 1344x768 (16:9)
  - "portrait": 768x1344 (9:16)
- `height`: Image height (set automatically based on aspect_ratio)
- `width`: Image width (set automatically based on aspect_ratio)
- `num_inference_steps`: Number of denoising steps (default: 60)
- `guidance_scale`: Classifier-free guidance scale (default: 3.5)
- `seed`: Random seed for reproducibility

### ControlNet Parameters
- `depth_controlnet_conditioning_scale`: Depth control strength (0.0-1.0)
- `canny_controlnet_conditioning_scale`: Canny control strength (0.0-1.0)
- `blur_controlnet_conditioning_scale`: Blur control strength (0.0-1.0)
- `depth_control_guidance_start`: When to start depth control (0.0-1.0)
- `canny_control_guidance_start`: When to start canny control (0.0-1.0)
- `blur_control_guidance_start`: When to start blur control (0.0-1.0)
- `depth_control_guidance_end`: When to end depth control (0.0-1.0)
- `canny_control_guidance_end`: When to end canny control (0.0-1.0)
- `blur_control_guidance_end`: When to end blur control (0.0-1.0)

### Preprocessing Parameters
- `canny_low_threshold`: Lower threshold for Canny edge detection (default: 50)
- `canny_high_threshold`: Upper threshold for Canny edge detection (default: 200)
- `blur_kernel_size`: Gaussian blur kernel size, must be odd (default: 51)

### System Parameters
- `device`: Torch device (default: "cuda")
- `torch_dtype`: Torch data type ("bfloat16", "float16", "float32")
- `use_4bit_quantization`: Enable 4-bit quantization (default: false)
- `cache_dir`: Cache directory for models
- `local_files_only`: Only use local files (default: true)
- `offline_mode`: Run in offline mode (default: true)
- `log_level`: Logging level ("DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL")

## Example: Multi-Style Batch

```json
[
    {
        "prompt": "anime style, vibrant colors",
        "input_image": "./inputs/portrait.png",
        "output_dir": "outputs/styles",
        "final_output": "outputs/styles/anime.png",
        "canny_controlnet_conditioning_scale": 0.9,
        "num_inference_steps": 50,
        "seed": 42
    },
    {
        "prompt": "oil painting style, classical",
        "input_image": "./inputs/portrait.png",
        "output_dir": "outputs/styles",
        "final_output": "outputs/styles/oil_painting.png",
        "canny_controlnet_conditioning_scale": 0.7,
        "depth_controlnet_conditioning_scale": 0.3,
        "num_inference_steps": 60,
        "seed": 42
    },
    {
        "prompt": "watercolor style, soft edges",
        "input_image": "./inputs/portrait.png",
        "output_dir": "outputs/styles",
        "final_output": "outputs/styles/watercolor.png",
        "canny_controlnet_conditioning_scale": 0.5,
        "blur_controlnet_conditioning_scale": 0.2,
        "blur_kernel_size": 31,
        "num_inference_steps": 40,
        "seed": 42
    }
]
```

## Processing Flow

1. The pipeline and models are loaded once at startup
2. For each configuration in the array:
   - Load the input image
   - Generate depth, canny, and blur control maps
   - Generate the final image using the control maps
   - Save all outputs to specified paths
3. Report success/failure statistics at the end

## Memory requirements: 

Depends on your model. Should just fit on an L40 or A40. (48GB mem needed)
```
depth-anything/Depth-Anything-V2-Base-hf
|   0  NVIDIA A40                     On  |   00000000:53:00.0 Off |                    0 |
|  0%   54C    P0            302W /  300W |   45243MiB /  46068MiB |    100%      Default |
```

`depth-anything/Depth-Anything-V2-Large-hf` gives a better quality depth but is too big to fit on 48gb with everything else. 


## Tips

- Keep all configurations in a batch using the same model paths for efficiency
- Use consistent output directory structure for easy organization
- Set specific seeds for reproducible results
- Adjust control scales to balance between prompt adherence and structural control