Add CUDA repository
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb sudo dpkg -i cuda-keyring_1.1-1_all.deb sudo apt-get update

Install CUDA toolkit WITHOUT driver components
sudo apt-get install -y cuda-toolkit-12-8 --no-install-recommends

# Essentials
```sudo apt install build-essential -y```

# Add CUDA repository keyring

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update

# Install CUDA toolkits WITHOUT driver packages
```sudo apt-get install -y --no-install-recommends nvidia-cuda-toolkit
sudo apt-get install -y --no-install-recommends cuda-toolkit-12-8```
