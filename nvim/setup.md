# neovim (lazyvim) setup
- (Manually) install nvim since the version in debian repos is often too old for lazyvim
- Export the binary to path by adding to `.bashrc` / `.zshrc`. For example, if the extracted nvim folder is in `/home/martin/custom/`:
```
export PATH="${PATH}:/home/martin/custom/nvim-linux-x86_64/bin"
```
- To be able to run a manually installed nvim as root (with sudo), one needs to create a symbolic link:
```bash
sudo ln -s $(which nvim) /usr/local/bin/nvim
```

- install lazyvim according to its own instructions

- to make sure everything works its best, better to have most common packages installed:
```bash
sudo apt install nodejs npm python3-pip python3-venv build-essential unzip tar curl
```

- to make sure yanking, copying and pasting works correctly between nvim and the outside:
```bash
sudo apt install xclip wl-clipboard
```

