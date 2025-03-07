# Asynchronous Python API for Synology DSM

[![](https://github.com/mib1185/py-synologydsm-api/workflows/Tests/badge.svg)](https://github.com/mib1185/py-synologydsm-api/actions?query=workflow%3ATests+branch%3Amaster)
[![Library version](https://img.shields.io/pypi/v/py-synologydsm-api.svg)](https://pypi.org/project/py-synologydsm-api)
[![Supported versions](https://img.shields.io/pypi/pyversions/py-synologydsm-api.svg)](https://pypi.org/project/py-synologydsm-api)
[![Downloads](https://pepy.tech/badge/py-synologydsm-api)](https://pypi.org/project/py-synologydsm-api)
[![Formated with Black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

# Installation

```bash
[sudo] pip install py-synologydsm-api
```

# Usage

You can import the module as `synology_dsm`.

## Constructor

```python
SynologyDSM(
    session,
    dsm_ip,
    dsm_port,
    username,
    password,
    use_https=False,
    timeout=10,
    device_token=None,
    debugmode=False,
)
```

For `session` a valid `aiohttp.ClientSession` needs to be provided. If ssl verification should be truned off, configure the session accordingly (eq. `aiohttp.ClientSession(connector=aiohttp.TCPConnector(verify_ssl=False)`)

`device_token` should be added when using a two-step authentication account, otherwise DSM will ask to login with a One Time Password (OTP) and requests will fail (see the login section for more details).

Default `timeout` is 10 seconds. `SynologyDSM` also takes a `aiohttp.ClientTimeout` as `timeout`.

## Login

Use the `login()` function to authenticate first, otherwise a `SynologyDSMNotLoggedInException` is raised.

It will return a boolean if it successed or faild to authenticate to DSM.

If your account need a two-step authentication (2SA), `login()` will raise `SynologyDSMLogin2SARequiredException`.
Call the function again with a One Time Password (OTP) as parameter, like `login("123456")` (better to be a string to handle first zero).
Store the `device_token` property so that you do not need to reconnect with password the next time you open a new `SynologyDSM` session.

## Code example

Every API has an `update()` function that is needed to get the first data, then the data is cached and updated at the next `update()` call.

The `SynologyDSM` class can also `update()` all APIs at once.

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM

async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)

async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()

    print("=== Information ===")
    await api.information.update()
    print("Model:           " + str(api.information.model))
    print("RAM:             " + str(api.information.ram) + " MB")
    print("Serial number:   " + str(api.information.serial))
    print("Temperature:     " + str(api.information.temperature) + " °C")
    print("Temp. warning:   " + str(api.information.temperature_warn))
    print("Uptime:          " + str(api.information.uptime))
    print("Full DSM version:" + str(api.information.version_string))
    print("Major DSM version:" + str(api.information.awesome_version.major))
    print("Minor DSM version:" + str(api.information.awesome_version.minor))
    print("Micro DSM version:" + str(api.information.awesome_version.micro))
    print("Patch DSM version:" + str(api.information.awesome_version.patch))
    print("--")

    print("=== Utilisation ===")
    await api.utilisation.update()
    print("CPU Load:        " + str(api.utilisation.cpu_total_load) + " %")
    print("Memory Use:      " + str(api.utilisation.memory_real_usage) + " %")
    print("Net Up:          " + str(api.utilisation.network_up()))
    print("Net Down:        " + str(api.utilisation.network_down()))
    print("--")

    print("=== Storage ===")
    await api.storage.update()
    for volume_id in api.storage.volumes_ids:
        print("ID:          " + str(volume_id))
        print("Status:      " + str(api.storage.volume_status(volume_id)))
        print("% Used:      " + str(api.storage.volume_percentage_used(volume_id)) + " %")
        print("--")

    for disk_id in api.storage.disks_ids:
        print("ID:          " + str(disk_id))
        print("Name:        " + str(api.storage.disk_name(disk_id)))
        print("S-Status:    " + str(api.storage.disk_smart_status(disk_id)))
        print("Status:      " + str(api.storage.disk_status(disk_id)))
        print("Temp:        " + str(api.storage.disk_temp(disk_id)))
        print("--")

    print("=== Shared Folders ===")
    await api.share.update()
    for share_uuid in api.share.shares_uuids:
        print("Share name:        " + str(api.share.share_name(share_uuid)))
        print("Share path:        " + str(api.share.share_path(share_uuid)))
        print("Space used:        " + str(api.share.share_size(share_uuid, human_readable=True)))
        print("Recycle Bin Enabled: " + str(api.share.share_recycle_bin(share_uuid)))
        print("--")

if __name__ == "__main__":
    asyncio.run(main())
```

## Download Station usage

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM

async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)

async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()

    if "SYNO.DownloadStation.Info" in api.apis:

        await api.download_station.get_info()
        await api.download_station.get_config()

        # The download list will be updated after each of the following functions:
        # You should have the right on the (default) directory that the download will be saved, or you will get a 403 or 406 error
        await api.download_station.create("http://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4")
        await api.download_station.pause("dbid_1")
        # Like the other function, you can eather pass a str or a list
        await api.download_station.resume(["dbid_1", "dbid_2"])
        await api.download_station.delete("dbid_3")

        # Manual update
        await api.download_station.update()

if __name__ == "__main__":
    asyncio.run(main())
```

## File Station usage

### List folders and files in specific folder

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM

async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)

async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()

    shared_folders = await api.file.get_shared_folders()
    for folder in shared_folders:
        print(f"############### {folder.name} ###############")
        print(f"path: {folder.path}")
        print(f"freespace: {folder.additional.volume_status.freespace}")
        print(f"totalspace: {folder.additional.volume_status.totalspace}")
        print(f"readonly: {folder.additional.volume_status.readonly}")

    files = await api.file.get_files(path="/home")
    for file in files:
        print(f"path: {file.path}")
        print(f"size: {file.additional.size}")
        print(f"is dir: {file.is_dir}")
        print(f"owner user: {file.additional.owner.user}")

if __name__ == "__main__":
    asyncio.run(main())
```

### Upload, Download and Delete files

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM

async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)

async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()

    # upload file direct from local
    await api.file.upload_file(path="/home", filename="myfile.name", source="/workspace/myfile.name")

    # upload file direct from local, create parent folder(s) if none exist.
    await api.file.upload_file(path="/home/new_folder", filename="myfile.name", source="/workspace/myfile.name", create_parents=True)

    # upload file from a stream reader
    await api.file.upload_file(path="/home", filename="myfile.name", source=open("/workspace/myfile.name", "rb"))

    # upload file from an AsyncIterator[bytes]
    loop = asyncio.get_running_loop()
    async def send_backup() -> AsyncIterator[bytes]:
        f = await loop.run_in_executor(None, open, "/workspace/myfile.name", "rb")
        try:
            while chunk := await loop.run_in_executor(None, f.read, 2**20):
                yield chunk
        finally:
            await loop.run_in_executor(None, f.close)

    async def open_backup() -> AsyncIterator[bytes]:
        return send_backup()
    await api.file.upload_file(path="/home", filename="myfile.name", source=await open_backup())

    # download file direct to local
    await api.file.download_file(path="/home", filename="myfile.name", target_file="/tmp/download.file")

    # download file via stream reader
    stream_reader = await api.file.download_file(path="/home", filename="myfile.name")
    with open("/tmp/download.file", "wb") as fh:
        async for data in stream_reader.iter_chunked(8192):
            fh.write(data)

    await api.file.delete_file(path="/home", filename="myfile.name")

if __name__ == "__main__":
    asyncio.run(main())
```

## External USB storage usage

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM

async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)

async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()

    await api.external_usb.update()
    for device in api.external_usb.get_devices.values():
        print("Name:                   " + str(device.device_name))
        print("Size:                   " + str(device.device_size_total(human_readable=True))
        print("Size in mb:             " + str(device.device_size_total())
        print("Partitions size total:  " + str(device.partitions_all_size_total(human_readable=True)))
        print("Partitions size used:   " + str(device.partitions_all_size_used(human_readable=True)))
        print("Partitions % Used:      " + str(device.partitions_all_percentage_used ) + " %")
        print("    === Partitions ===")
        for part in device.device_partitions.values():
            print("    Share name:             " + str(part.share_name))
            print("    Filesystem:             " + str(part.filesystem))
            print("    Size:                   " + str(part.partition_size_total(human_readable=True)))
            print("    Used:                   " + str(part.partition_size_used(human_readable=True)))
            print("    % Used:                 " + str(part.partition_percentage_used) + " %")

if __name__ == "__main__":
    asyncio.run(main())
```

## Photos usage

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM

async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)

async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()

    albums = await api.photos.get_albums()

    # download each item of each album
    for album in albums:
        print(f"'{album.name}' has {album.item_count} items")

        items = await api.photos.get_items_from_album(album)
        for item in items:
            print(f"{item.file_name}")
            with open(item.file_name, "wb") as fh:
                fh.write(await api.photos.download_item(item))

    # search for items and download them
    items = await api.photos.get_items_from_search("pinky and the brain")
    for item in items:
        with open(item.file_name, "wb") as fh:
            fh.write(await api.photos.download_item(item))

if __name__ == "__main__":
    asyncio.run(main())
```

## Surveillance Station usage

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM

async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)

async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()

    surveillance = api.surveillance_station
    await surveillance.update() # First update is required

    # Returns a list of cached cameras available
    cameras = surveillance.get_all_cameras()

    # Assuming there's at least one camera, get the first camera_id
    camera_id = cameras[0].camera_id

    # Returns cached camera object by camera_id
    camera = surveillance.get_camera(camera_id)

    # Returns cached motion detection enabled
    motion_setting = camera.is_motion_detection_enabled

    # Return bytes of camera image
    await surveillance.get_camera_image(camera_id)

    # Updates all cameras/motion settings and cahce them
    await surveillance.update()

    # Gets Home Mode status
    home_mode_status = await surveillance.get_home_mode_status()

    # Sets home mode - true is on, false is off
    await surveillance.set_home_mode(True)

if __name__ == "__main__":
    asyncio.run(main())
```

## System usage

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM

async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)

async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()

    system = api.system

    # Reboot NAS
    await system.reboot()

    # Shutdown NAS
    await system.shutdown()

    # Manual update system information
    await system.update()

    # Get CPU information
    system.cpu_clock_speed
    system.cpu_cores
    system.cpu_family
    system.cpu_series

    # Get NTP settings
    system.enabled_ntp
    system.ntp_server

    # Get system information
    system.firmware_ver
    system.model
    system.ram_size
    system.serial
    system.sys_temp
    system.time
    system.time_zone
    system.time_zone_desc
    system.up_time

    # Get list of all connected USB devices
    system.usb_dev

if __name__ == "__main__":
    asyncio.run(main())
```

## Upgrade usage

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM

async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)

async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()
    upgrade = api.upgrade

    # Manual update upgrade information
    await upgrade.update()

    # check if DSM update is available
    if upgrade.update_available:
        do something ...

    # get available version string (return None if no update available)
    upgrade.available_version

    # get need of reboot (return None if no update available)
    upgrade.reboot_needed

    # get need of service restarts (return None if no update available)
    upgrade.service_restarts

if __name__ == "__main__":
    asyncio.run(main())
```

## Virtual Machine Manager usage

### Show information about all existing guests

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM


async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)


async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()

    await api.virtual_machine_manager.update()

    guests = api.virtual_machine_manager.get_all_guests()

    for guest in guests:
        print(f"############### {guest.name} ###############")
        print(f"autorun: {guest.autorun}")
        print(f"description: {guest.description}")
        print(f"guest_id: {guest.guest_id}")
        print(f"status: {guest.status}")
        print(f"vcpu_num: {guest.vcpu_num}")
        print(f"vram_size: {guest.vram_size / 1024} MiBytes")
        print(f"host_cpu_usage: {guest.host_cpu_usage / 10} %")
        print(f"host_ram_usage: {round(guest.host_ram_usage / 1024,1)} MiBytes")


if __name__ == "__main__":
    asyncio.run(main())
```

### Perform power actions on guests

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM


async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(verify_ssl=False)
    ) as session:
        await do(session)


async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()

    await api.virtual_machine_manager.update()

    # start a guest
    await api.virtual_machine_manager.guest_poweron("{guest.guest_id}")

    # power off a guest
    await api.virtual_machine_manager.guest_poweroff("{guest.guest_id}")

    # graceful shutdown a guest (needs working guest-agent, else it is not graceful)
    await api.virtual_machine_manager.guest_shutdown("{guest.guest_id}")

    # graceful restart a guest (needs working guest-agent, else it is not graceful)
    await api.virtual_machine_manager.guest_restart("{guest.guest_id}")

if __name__ == "__main__":
    asyncio.run(main())
```

## Hyperbackup usage

```python
import asyncio
import aiohttp
from synology_dsm import SynologyDSM

async def main():
    print("Creating Valid API")
    async with aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(ssl=False)
    ) as session:
        await do(session)

async def do(session: aiohttp.ClientSession):
    api = SynologyDSM(session, "<IP/DNS>", "<port>", "<username>", "<password>")
    await api.login()
    print("=== Hyperbackup ===")
    hyperbackup = api.hyperbackup
    await hyperbackup.update()

    for task in hyperbackup.get_all_tasks():
        print("task_id:                 " + str(task.task_id))
        print("name                     " + str(task.name))
        print("status:                  " + str(task.status))
        print("health:                  " + str(task.health))
        print("is_backing_up            " + str(task.is_backing_up))
        print("backup_progress          " + str(task.backup_progress))
        print("state                    " + str(task.state))
        print("target_online            " + str(task.target_online))
        print("used_size                " + str(task.used_size(True)))
        print("previous_result          " + str(task.previous_result))
        print("has_schedule             " + str(task.has_schedule))
        print("next_backup_time         " + str(task.next_backup_time))
        print("previous_backup_time     " + str(task.previous_backup_time))
        print("previous_backup_end_time " + str(task.previous_backup_end_time))
        print("previous_success_time    " + str(task.previous_success_time))
        print("--")

if __name__ == "__main__":
    asyncio.run(main())
```

# Credits / Special Thanks

- [@florianeinfalt](https://github.com/florianeinfalt)
- [@tchellomello](https://github.com/tchellomello)
- [@Quentame](https://github.com/Quentame) (Multiple API addition & tests)
- [@aaska](https://github.com/aaska) (DSM 5 tests)
- [@chemelli74](https://github.com/chemelli74) (2SA tests)
- [@snjoetw](https://github.com/snjoetw) (Surveillance Station library)
- [@shenxn](https://github.com/shenxn) (Surveillance Station tests)
- [@Gestas](https://github.com/Gestas) (Shared Folders)
- [@lodesmets](https://github.com/lodesmets) (Synology Photos)

Found Synology API "documentation" on this repo : https://github.com/kwent/syno/tree/master/definitions

# Official references

- [Calendar API documentation (2015-2019)](https://global.download.synology.com/download/Document/Software/DeveloperGuide/Package/Calendar/2.4/enu/Synology_Calendar_API_Guide_enu.pdf)
- [Download Station API documentation (2012-2014)](https://global.download.synology.com/download/Document/Software/DeveloperGuide/Package/DownloadStation/All/enu/Synology_Download_Station_Web_API.pdf)
- [File Station API documentation (2013-2019)](https://global.download.synology.com/download/Document/Software/DeveloperGuide/Package/FileStation/All/enu/Synology_File_Station_API_Guide.pdf)
- [Surveillance Station API documentation (2012-2020)](https://global.download.synology.com/download/Document/Software/DeveloperGuide/Package/SurveillanceStation/All/enu/Surveillance_Station_Web_API.pdf)
- [Virtual Machine Manager API documentation (2015-2019)](https://global.download.synology.com/download/Document/Software/DeveloperGuide/Package/Virtualization/All/enu/Synology_Virtual_Machine_Manager_API_Guide.pdf)
