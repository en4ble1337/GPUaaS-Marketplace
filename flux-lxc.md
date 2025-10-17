Add CUDA repository
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb sudo dpkg -i cuda-keyring_1.1-1_all.deb sudo apt-get update

Install CUDA toolkit WITHOUT driver components
sudo apt-get install -y cuda-toolkit-12-8 --no-install-recommends
