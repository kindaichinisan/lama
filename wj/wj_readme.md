# Dockerfile info

# 1 Build the docker image only. Not to run it
cd docker
docker build --network=host -f Dockerfile_Orin_NX -t lama:jetson .

# 2 Run the docker image as a container (wrong)
cd docker
docker run --rm -it --runtime nvidia --network host -v $(pwd):/workspace/lama -w /workspace/lama lama:jetson

# 3 Run the docker image. The folder is linked to host folder. Edit on host folder for GUI editing and test on container.
docker run --rm -it --runtime nvidia --network host -v ~/WJ_git/lama:/workspace/lama lama:jetson

# 4 Check GPU
python3 -c "import torch; print('Torch:', torch.__version__); print('CUDA:', torch.cuda.is_available()); print('GPU:', torch.cuda.get_device_name(0))"

# 5 Issues
## 5.1 Your LaMa code expects old Albumentations APIs
python3 -m pip uninstall -y albumentations
python3 -m pip install --no-cache-dir     --index-url https://pypi.org/simple     "albumentations==0.5.2"

## 5.2 numpy is too new
NumPy is 2.x.
imgaug, which Albumentations 0.5.2 uses, expects the old NumPy API:
np.sctypes
but NumPy 2.0 removed it.

python3 -m pip uninstall -y numpy
python3 -m pip install --no-cache-dir \
    --index-url https://pypi.org/simple \
    "numpy==1.26.4"

## 5.3 scipy is too new
python3 -m pip uninstall -y scipy
python3 -m pip install --no-cache-dir \
    --index-url https://pypi.org/simple \
    "scipy==1.11.4"

## 5.4 can run but PyTorch 2.6+ checkpoint loading compatibility issue
track_running_stats=True) (32): ReLU(inplace=True) (33): ReflectionPad2d((3, 3, 3, 3)) (34): Conv2d(64, 3, kernel_size=(7, 7), stride=(1, 1)) (35): Sigmoid() ) ) [2026-08-21 09:42:38,286][saicinpainting.training.trainers.base][INFO] - BaseInpaintingTrainingModule init done [2026-08-21 09:42:38,827][__main__][CRITICAL] - Prediction failed due to Weights only load failed. This file can still be loaded, to do so you have two options, do those steps only if you trust the source of the checkpoint. (1) In PyTorch 2.6, we changed the default value of the weights_only argument in torch.load from False to True. Re-running torch.load with weights_only set to False will likely succeed, but it can result in arbitrary code execution. Do it only if you got the file from a trusted source. (2) Alternatively, to load with weights_only=True please check the recommended steps in the following error message. WeightsUnpickler error: Unsupported global: GLOBAL pytorch_lightning.callbacks.model_checkpoint.ModelCheckpoint was not an allowed global by default. Please use torch.serialization.add_safe_globals([pytorch_lightning.callbacks.model_checkpoint.ModelCheckpoint]) or the torch.serialization.safe_globals([pytorch_lightning.callbacks.model_checkpoint.ModelCheckpoint]) context manager to allowlist this global if you trust this class/function. Check the documentation of torch.load to learn more about types accepted by default with weights_only https://pytorch.org/docs/stable/generated/torch.load.html.: Traceback (most recent call last): File "/workspace/lama/bin/predict.py", line 58, in main model = load_checkpoint(train_config, checkpoint_path, strict=False, map_location='cpu') ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ File "/workspace/lama/saicinpainting/training/trainers/__init__.py", line 27, in load_checkpoint state = torch.load(path, map_location=map_location) ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ File "/opt/venv/lib/python3.12/site-packages/torch/serialization.py", line 1524, in load raise pickle.UnpicklingError(_get_wo_message(str(e))) from None _pickle.UnpicklingError: Weights only load failed. This file can still be loaded, to do so you have two options, do those steps only if you trust the source of the checkpoint. (1) In PyTorch 2.6, we changed the default value of the weights_only argument in torch.load from False to True. Re-running torch.load with weights_only set to False will likely succeed, but it can result in arbitrary code execution. Do it only if you got the file from a trusted source. (2) Alternatively, to load with weights_only=True please check the recommended steps in the following error message. WeightsUnpickler error: Unsupported global: GLOBAL pytorch_lightning.callbacks.model_checkpoint.ModelCheckpoint was not an allowed global by default. Please use torch.serialization.add_safe_globals([pytorch_lightning.callbacks.model_checkpoint.ModelCheckpoint]) or the torch.serialization.safe_globals([pytorch_lightning.callbacks.model_checkpoint.ModelCheckpoint]) context manager to allowlist this global if you trust this class/function. Check the documentation of torch.load to learn more about types accepted by default with weights_only https://pytorch.org/docs/stable/generated/torch.load.html.

Modify /workspace/lama/saicinpainting/training/trainers/__init__.py from
`state = torch.load(path, map_location=map_location)`
to 
`state = torch.load(path, map_location=map_location, weights_only=False)`

## 5.5 Process killed.
Attempt to execute tegrastats in container using another Terminal in host PC: docker exec -it <containerid> bash  
Use tegrastats in host PC to check. Cannot do it in docker container as tegrastats is a Jetson host utility, and it isn't necessarily installed/exposed inside your Docker container.  

8133mW/3098mW VDD_SOC 2079mW/1930mW 08-21-2026 18:23:23 RAM 6567/15656MB (lfb 54x4MB) SWAP 3109/7828MB (cached 5MB) CPU [31%@1984,54%@1984,50%@1984,64%@1984,54%@1984,55%@1984,55%@1984,100%@1984] GR3D_FREQ 7% cv0@57.281C cpu@58.937C soc2@54.687C soc0@55.125C cv1@55.437C gpu@54.375C tj@58.937C soc1@56.375C cv2@54.656C VDD_IN 12673mW/8213mW VDD_CPU_GPU_CV 7050mW/3122mW VDD_SOC 2121mW/1931mW 08-21-2026 18:23:24 RAM 6238/15656MB (lfb 65x4MB) SWAP 3109/7828MB (cached 5MB) CPU [23%@1984,9%@1984,20%@1984,10%@1984,7%@1984,7%@1984,8%@1984,100%@1984] GR3D_FREQ 7% cv0@54.375C cpu@56.531C soc2@53.906C soc0@55.062C cv1@54.062C gpu@54C tj@56.531C soc1@56.281C cv2@54.343C VDD_IN 8820mW/8217mW VDD_CPU_GPU_CV 3159mW/3123mW VDD_SOC 2175mW/1933mW 08-21-2026 18:23:25 RAM 9686/15656MB (lfb 19x4MB) SWAP 3109/7828MB (cached 5MB) CPU [33%@1984,5%@1984,7%@1984,3%@1984,6%@1984,7%@1984,11%@1984,100%@1984] GR3D_FREQ 31% cv0@54.312C cpu@56.093C soc2@53.812C soc0@55C cv1@53.875C gpu@53.75C tj@56.125C soc1@56.125C cv2@54.281C VDD_IN 8596mW/8219mW VDD_CPU_GPU_CV 2803mW/3121mW VDD_SOC 2175mW/1934mW 08-21-2026 18:23:26 RAM 12591/15656MB (lfb 119x1MB) SWAP 3109/7828MB (cached 5MB) CPU [53%@1984,8%@1984,8%@1984,7%@1984,4%@1984,0%@1984,9%@1984,100%@1984] GR3D_FREQ 7% cv0@54.25C cpu@56.031C soc2@53.875C soc0@54.843C cv1@53.625C gpu@53.781C tj@56.093C soc1@56.093C cv2@54.093C VDD_IN 8701mW/8222mW VDD_CPU_GPU_CV 2922mW/3119mW VDD_SOC 2175mW/1935mW 08-21-2026 18:23:27 RAM 14305/15656MB (lfb 32x1MB) SWAP 3108/7828MB (cached 5MB) CPU [26%@1984,9%@1984,22%@1984,42%@1984,1%@1984,19%@1984,2%@1984,100%@1984] GR3D_FREQ 7% cv0@53.937C cpu@56.031C soc2@53.468C soc0@54.781C cv1@53.593C gpu@53.562C tj@56.031C soc1@55.875C cv2@54C VDD_IN 8477mW/8223mW VDD_CPU_GPU_CV 2961mW/3119mW VDD_SOC 2096mW/1936mW 08-21-2026 18:23:28 RAM 14897/15656MB (lfb 31x1MB) SWAP 3316/7828MB (cached 5MB) CPU [16%@729,52%@729,27%@729,5%@729,47%@1984,6%@1984,6%@1984,99%@1984] GR3D_FREQ 6% cv0@54.656C cpu@56.156C soc2@53.593C soc0@54.812C cv1@53.75C gpu@53.406C tj@56.156C soc1@55.968C cv2@54.031C VDD_IN 8517mW/8225mW VDD_CPU_GPU_CV 3316mW/3120mW VDD_SOC 1977mW/1937mW 08-21-2026 18:23:29 RAM 15093/15656MB (lfb 1x2MB) SWAP 3618/7828MB (cached 4MB) CPU [13%@729,12%@729,24%@729,28%@729,63%@1984,46%@1984,4%@1984,99%@1984] GR3D_FREQ 13% cv0@54.125C cpu@55.906C soc2@53.437C soc0@54.843C cv1@53.937C gpu@53.656C tj@56.031C soc1@56.031C cv2@54.031C VDD_IN 8557mW/8227mW VDD_CPU_GPU_CV 3395mW/3121mW VDD_SOC 1977mW/1937mW 08-21-2026 18:23:30 RAM 15179/15656MB (lfb 31x1MB) SWAP 4161/7828MB (cached 4MB) CPU [31%@1984,64%@1984,43%@1984,10%@1984,17%@1984,25%@1984,2%@1984,100%@1984] GR3D_FREQ 37% cv0@54.406C cpu@55.968C soc2@53.625C soc0@54.687C cv1@53.75C gpu@53.687C tj@56C soc1@56C cv2@54.062C VDD_IN 8701mW/8230mW VDD_CPU_GPU_CV 3553mW/3124mW VDD_SOC 1977mW/1937mW 08-21-2026 18:23:31 RAM 15262/15656MB (lfb 56x1MB) SWAP 4807/7828MB (cached 4MB) CPU [46%@960,31%@960,24%@960,28%@960,55%@1984,24%@1984,51%@1984,53%@1984] GR3D_FREQ 4% cv0@54.781C cpu@56.062C soc2@53.468C soc0@54.656C cv1@53.968C gpu@53.468C tj@56.062C soc1@55.812C cv2@54C VDD_IN 8701mW/8233mW VDD_CPU_GPU_CV 3593mW/3127mW VDD_SOC 1977mW/1937mW 08-21-2026 18:23:32 RAM 15251/15656MB (lfb 67x1MB) SWAP 5538/7828MB (cached 4MB) CPU [24%@1984,13%@1984,88%@1984,33%@1984,29%@1984,10%@1984,97%@1984,4%@1984] GR3D_FREQ 17% cv0@54C cpu@55.968C soc2@53.5C soc0@54.656C cv1@53.593C gpu@53.968C tj@55.968C soc1@55.75C cv2@54C VDD_IN 8517mW/8234mW VDD_CPU_GPU_CV 3435mW/3129mW VDD_SOC 1938mW/1937mW 08-21-2026 18:23:33 RAM 15254/15656MB (lfb 62x1MB) SWAP 6151/7828MB (cached 4MB) CPU [9%@1984,25%@1984,98%@1984,2%@1984,5%@1984,5%@1984,97%@1984,3%@1984] GR3D_FREQ 31% cv0@54C cpu@56.062C soc2@53.437C soc0@54.687C cv1@53.531C gpu@53.406C tj@56.062C soc1@55.781C cv2@53.968C VDD_IN 8319mW/8235mW VDD_CPU_GPU_CV 3277mW/3129mW VDD_SOC 1938mW/1937mW 08-21-2026 18:23:34 RAM 15309/15656MB (lfb 35x1MB) SWAP 6772/7828MB (cached 4MB) CPU [11%@1984,28%@1984,97%@1984,29%@1984,39%@1651,23%@1651,54%@1651,7%@1651] GR3D_FREQ 19% cv0@54.281C cpu@56C soc2@53.437C soc0@54.781C cv1@53.75C gpu@53.437C tj@56C soc1@55.937C cv2@54.031C VDD_IN 8596mW/8237mW VDD_CPU_GPU_CV 3474mW/3131mW VDD_SOC 1977mW/1938mW 08-21-2026 18:23:35 RAM 15339/15656MB (lfb 40x1MB) SWAP 7416/7828MB (cached 4MB) CPU [22%@1984,79%@1984,10%@1984,29%@1984,47%@1984,9%@1984,5%@1984,44%@1984] GR3D_FREQ 6% cv0@54.562C cpu@55.718C soc2@53.5C soc0@54.531C cv1@53.718C gpu@53.281C tj@55.843C soc1@55.843C cv2@53.968C VDD_IN 8477mW/8238mW VDD_CPU_GPU_CV 3316mW/3132mW VDD_SOC 1977mW/1938mW 08-21-2026 18:23:36 RAM 15451/15656MB (lfb 60x1MB) SWAP 7822/7828MB (cached 3MB) CPU [13%@1984,32%@1984,75%@1984,22%@1984,68%@1420,7%@1420,3%@1420,36%@1420] GR3D_FREQ 43% cv0@54.093C cpu@55.937C soc2@53.406C soc0@54.562C cv1@53.562C gpu@53.468C tj@55.937C soc1@55.656C cv2@53.843C VDD_IN 8438mW/8240mW VDD_CPU_GPU_CV 3316mW/3134mW VDD_SOC 1977mW/1938mW 08-21-2026 18:23:37 RAM 15031/15656MB (lfb 1x4MB) SWAP 7400/7828MB (cached 2MB) CPU [80%@1984,80%@1984,100%@1984,76%@1984,87%@1574,67%@1574,70%@1574,76%@1574] GR3D_FREQ 50% cv0@54.406C cpu@56.531C soc2@53.718C soc0@54.718C cv1@54C gpu@53.718C tj@56.531C soc1@55.812C cv2@54.093C VDD_IN 10187mW/8251mW VDD_CPU_GPU_CV 5077mW/3145mW VDD_SOC 1974mW/1938mW 08-21-2026 18:23:38 RAM 13136/15656MB (lfb 77x4MB) SWAP 4101/7828MB (cached 2MB) CPU [4%@1984,2%@1984,100%@1984,15%@1984,35%@729,3%@729,8%@729,8%@729] GR3D_FREQ 15% cv0@53.656C cpu@55.687C soc2@53.281C soc0@54.75C cv1@53.312C gpu@53.375C tj@55.781C soc1@55.781C cv2@54.031C VDD_IN 7725mW/8248mW VDD_CPU_GPU_CV 2847mW/3143mW VDD_SOC 1901mW/1938mW 08-21-2026 18:23:39 RAM 4299/15656MB (lfb 296x4MB) SWAP 3362/7828MB (cached 2MB) CPU [6%@729,3%@729,95%@729,34%@729,30%@1984,12%@1984,8%@1984,4%@1984] GR3D_FREQ 25% cv0@53.875C cpu@55.218C soc2@53.343C soc0@54.5C cv1@53.5C gpu@53.218C tj@55.562C soc1@55.562C cv2@53.687C VDD_IN 7618mW/8244mW VDD_CPU_GPU_CV 2689mW/3140mW VDD_SOC 1901mW/1938mW 08-21-2026 18:23:40 RAM 4305/15656MB (lfb 296x4MB) SWAP 3356/7828MB (cached 2MB) CPU [16%@1113,14%@1113,5%@1113,23%@1113,12%@1984,7%@1984,7%@1984,6%@1984] GR3D_FREQ 6% cv0@53.406C cpu@55.031C soc2@53.031C soc0@54.437C cv1@53.125C gpu@53.093C tj@55.531C soc1@55.531C cv2@53.593C VDD_IN 6319mW/8233mW VDD_CPU_GPU_CV 1545mW/3131mW VDD_SOC 1785mW/1937mW

memory exhaustion as LaMa started allocating memory and rapidly consumed essentially all 15.6 GB RAM.
At the peak:
RAM 15451 / 15656 MB
SWAP 7822 / 7828 MB

Using a small image like 512x512 can work.

## 5.6 folder created by container cannot be modified by host PC
docker run --rm -it --runtime nvidia --network host --user $(id -u):$(id -g) -v ~/WJ_git/lama:/workspace/lama lama:jetson

# 6 To run
python3 bin/predict.py     model.path=$(pwd)/big-lama     indir=$(pwd)/LaMa_test_images    outdir=$(pwd)/output

# 7 download data
❗️❗️❗️ All yandex dist links went bad, you can download the model from the google drive ❗️❗️❗️
LaMa_test_images: https://drive.google.com/drive/folders/1B2x7eQDgecTL0oh3LSIBDGj0fTxs6Ips