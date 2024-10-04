I am running this in a virtual machine because there seems to be some configuration issue in my host machine, which results in build failure. Follow these steps to build the radxa groundstation image for OpenIPC either in your host machine or in a virtual machine (I am using ubuntu 20 in virtual machine)

1. Setup env for rsdk by running these commands:
    ```sh
    sudo apt update
    sudo apt install git qemu-user-static binfmt-support curl docker.io -y
    sudo usermod -a -G docker $USER
    # Reboot for the above command to take affect
    sudo reboot
    ```

2. Setup rsdk:
    ```sh
    git clone --recurse-submodules https://github.com/RadxaOS-SDK/rsdk.git
    cd rsdk
    npm install @devcontainers/cli
    echo 'PATH="$PWD/src/bin:$PWD/node_modules/.bin:$PATH"' >> ~/.bashrc
    ```

3. Create an overlays directory
    ```sh
    mkdir overlays
    cd overlays
    ```

4. Clone JohhnGoblin's Radxa Image build script
    ```sh
    git clone https://github.com/JohnDGodwin/zero3w-gs.git
    #Move back to the rsdk root folder
    cd ..
    ```

5. Modify the file `src/share/rsdk/build/rootfs.jsonnet` to run JohhnGoblin's script during image build. Include the following lines inside the `customize-hooks` section
    ```sh
    'cp -r "overlays/zero3w-gs" "$1/"',
    'chroot "$1" chmod +x /zero3w-gs/install-gs.sh',
    'chroot "$1" sh -c "cd /zero3w-gs && ./install-gs.sh"',
    ```

customize-hooks section should now look like this:

    "customize-hooks"+:
        [
            'echo "127.0.1.1	%(product)s" >> "$1/etc/hosts"' % { product: product },
            'cp "%(output_dir)s/config.yaml" "$1/etc/rsdk/"' % { output_dir: output_dir },
            'echo "FINGERPRINT_VERSION=\'2\'" > "$1/etc/radxa_image_fingerprint"',
            'echo "RSDK_BUILD_DATE=\'$(date -R)\'" >> "$1/etc/radxa_image_fingerprint"',
            'echo "RSDK_REVISION=\'%(rsdk_rev)s\'" >> "$1/etc/radxa_image_fingerprint"' % { rsdk_rev: rsdk_rev },
            'echo "RSDK_CONFIG=\'/etc/rsdk/config.yaml\'" >> "$1/etc/radxa_image_fingerprint"',
            'chroot "$1" update-initramfs -cvk all',
            'cp -r "overlays/zero3w-gs" "$1/"',
            'chroot "$1" chmod +x /zero3w-gs/install-gs.sh',
            'chroot "$1" sh -c "cd /zero3w-gs && ./install-gs.sh"',
            |||
                mkdir -p "%(output_dir)s/seed"
                cp "$1/etc/radxa_image_fingerprint" "%(output_dir)s/seed"
                cp "$1/etc/rsdk/"* "%(output_dir)s/seed"
                tar Jvcf "%(output_dir)s/seed.tar.xz" -C "%(output_dir)s/seed" .
                rm -rf "%(output_dir)s/seed"
            ||| % { output_dir: output_dir },
        ]
   

6. Now you can follow the steps to start the devcontainer and build the image
    ```sh
    rsdk devcon up
    rsdk devcon
    rsdk shell
    rsdk build radxa-zero3 bullseye cli
    ```

7. This will create an output.img file inside `out/radxa-zero3_bullseye_cli` folder

8. This output.img file can now be flashed into an SD card and can be used to boot radxa.


> **Note:**
> - During the build process, some popup appears for which you need to select the OK button. We will have to find a way to bypass those if we plan to move this build process to CI/CD pipelines.
> - Currently building and installing of the AU/EU adapter driver modules and wifibroadcast is failing