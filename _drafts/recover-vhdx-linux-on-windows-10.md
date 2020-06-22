---
layout: post
title: Recover vhdx linux on Windows 10
cover-img: https://miro.medium.com/max/700/0*iuVrmoYNMxFRdNl6
tags: [books, test]
---
When I upgraded my Windows 10 to use the new WSL2 subsystem with Docker I lost all my volumes from the previuos docker installation.

The previous docker installation was installed on Hyper-V. I was left with a 37GB `DockerDesktop.vhdx` file that I could not mount to disk manager.

Steps taken to get the files:

1. Use PowerShell as Administrator to convert the `vhdx` file to `vhd` using command:
`Convert-vhd -Path .\DockerDesktop.vhdx -DestinationPath .\DockerDesktop.vhd -VHDType fixed`
2. Use 7-Zip to open the new `DockerDesktop.vhd` file.
3. The volumes will be archived at `/docker/volumes` location in the archive
4. Copy the volume folders to your safe location
5. To import the old data into new volumes you will have to follow this steps:
- crate a new docker volume `docker volume create mynewvolume`
- create a dummy container that will mount the newly created volume, example: `docker container create --name dummy -v mynewvolume:/newvolume hello-world`
- copy the old files to the new volume: `docker cp /location/to/myoldfiles/.  dummy:/newvolume/`
- cleanup the dummy container: `docker rm dummy`
