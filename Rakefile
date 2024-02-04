
namespace :plex do

  task :mkdirs do
    # kenneth@fado ~/git/pms-docker (fado) $ sudo mkdir /media/plex
    # kenneth@fado ~/git/pms-docker (fado) $ sudo mkdir /media/plex/{database,transcode}
    # kenneth@fado ~/git/pms-docker (fado) $ sudo groupadd media 
    # kenneth@fado ~/git/pms-docker (fado) $ sudo usermod -aG media kenneth 
    # kenneth@fado ~/git/pms-docker (fado) $ sudo chgrp -R media /media/plex
    # kenneth@fado ~/git/pms-docker (fado) $ sudo chmod -R 0775 /media/plex 
    # kenneth@fado ~/git/pms-docker (fado) $ ls -lR /media
    # /media:
    # drwxr-xr-x 2 root root  4096 Nov 21 16:13 cdrom
    # drwxrwxr-x 4 root media 4096 Feb  4 11:35 plex
    # 
    # /media/plex:
    # drwxrwxr-x 2 root media 4096 Feb  4 11:35 database
    # drwxrwxr-x 2 root media 4096 Feb  4 11:35 transcode
  end

  task :mount do
    # [   59.324542] usb 1-3: new high-speed USB device number 2 using xhci_hcd
    # [   79.532542] usb 1-3: new high-speed USB device number 3 using xhci_hcd
    # [   79.769166] usb 1-3: New USB device found, idVendor=059f, idProduct=101c, bcdDevice= 0.00
    # [   79.769177] usb 1-3: New USB device strings: Mfr=1, Product=2, SerialNumber=3
    # [   79.769182] usb 1-3: Product: 2big quadra
    # [   79.769185] usb 1-3: Manufacturer: LaCie
    # [   79.769187] usb 1-3: SerialNumber: 00D04B6202047C6B
    # [   79.785445] input: LaCie 2big quadra as /devices/pci0000:00/0000:00:08.1/0000:04:00.3/usb1/1-3/1-3:1.1/0003:059F:101C.0003/input/input21
    # [   79.844894] hid-generic 0003:059F:101C.0003: input,hidraw2: USB HID v1.11 Device [LaCie 2big quadra] on usb-0000:04:00.3-3/input1
    # [   79.857230] SCSI subsystem initialized
    # [   79.858918] usb-storage 1-3:1.0: USB Mass Storage device detected
    # [   79.858987] scsi host0: usb-storage 1-3:1.0
    # [   79.859038] usbcore: registered new interface driver usb-storage
    # [   79.860088] usbcore: registered new interface driver uas
    # [   80.868385] scsi 0:0:0:0: Direct-Access     LaCie  2 BigQuadra        0    PQ: 0 ANSI: 4
    # [   80.871409] scsi 0:0:0:0: Attached scsi generic sg0 type 0
    # [   80.875734] sd 0:0:0:0: [sda] 3906929168 512-byte logical blocks: (2.00 TB/1.82 TiB)
    # [   80.877336] sd 0:0:0:0: [sda] Write Protect is off
    # [   80.877339] sd 0:0:0:0: [sda] Mode Sense: 10 00 00 00
    # [   80.878963] sd 0:0:0:0: [sda] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
    # [   80.907925]  sda: sda1
    # [   80.908292] sd 0:0:0:0: [sda] Attached SCSI disk

    # kenneth@fado ~/git/pms-docker (fado) $ sudo fdisk -l
    # Device     Boot Start        End    Sectors  Size Id Type
    # /dev/sda1          44 3906928607 3906928564  1.8T 83 Linux

    # kenneth@fado ~/git/pms-docker (fado) $ sudo mount -t ext4 /dev/sda1 /mnt/lacie
    # kenneth@fado ~/git/pms-docker (fado) $ ls /mnt/lacie
    # audio  backups  books  cdrip  downloads  encoding-queue  flac  lossless  lost+found  movies  music  netgear_downloader  phone  photos  README  tagger-queue  tmp  torrents  traders-ftp-root  tv
  end

  task :manual_run do
    # using network=host mode means plex container will bind to ports directly
    # on host ip (if fado is 192.168.1.9, 192.168.1.9:32400 will be ADVERTISE_IP)
    # and plex will automatically regard all 192.168.1.* ips as being on LAN, &
    # automatically use higher quality defaults for local clients

    # do we need to re-register w/new PLEX_CLAIM every reboot?
    # https://www.plex.tv/claim/
    PLEX_CLAIM = ENV["PLEX_CLAIM"]  # this is now hardcoded in docker-compose.fado.yml
    # https://github.com/kennethd/pms-docker/blob/master/README.md
    cmd = [
      "docker run -d" \
      "--name plex" \
      "--network=host" \
      "-e TZ=\"America/New_York\"" \
      "-e PLEX_CLAIM=\"#{ENV['PLEX_CLAIM']}\"" \
      "-v plex-database:/config" \
      "-v plex-transcode-temp:/transcode" \
      "-v /mnt/lacie/tv:/data/tv" \
      "-v /mnt/lacie/movies:/data/movies" \
      "-v /mnt/lacie/music:/data/music" \
      "plexinc/pms-docker"
    ].join(" ")
    sh(cmd)
  end

  desc "bring up docker container

  assumes you have docker-compose.yml in current directory, or a symlink:

    docker-compose.yml -> docker-compose.fado.yml
  "
  task :up do
    sh("docker-compose up -d")
  end
end
