# Catalyst Center SWIM: Bruno collection (2.3.7.x)

Classic SWIM flow over the Catalyst Center Intent API, with selected ports shut before the upgrade and re-enabled after it.

## Setup

1. In Bruno: *Open Collection*, pick this folder.
2. Select the `lab` environment and fill in `baseUrl`, `username`, `password` (secret), `hostname`, `imageName`
   and `ports` (comma-separated full interface names, as Catalyst Center shows them).
3. If Catalyst Center uses an internal CA: Preferences > General, either add the CA certificate or turn off SSL/TLS verification for the lab.

## Order of play

| Step | Request | Notes |
|---|---|---|
| 1 | 01 Auth / Get token | sets `token` |
| 2 | 02 Discovery / Get device by hostname, Get image by name | sets `deviceUuid`, `imageUuid`, `deviceFamilyId` |
| 3 | 03 Import / Import image from URL | only if the image is not in the repo |
| 4 | 04 Golden / Tag image as golden | optional |
| 5 | 06 Upgrade / Distribute image, then 09 Tasks / Get task | copy to flash before the window, no impact |
| 6 | 05 Ports down / Snapshot ports | records state into `portSnapshot` |
| 7 | 05 Ports down / Shutdown next port | needs `allowPortChanges: true`; re-send until queue empty |
| 8 | 10 Checks / Run show commands, 09 Tasks / Get task, 10 Checks / Get command output | confirm ports are down on the device |
| 9 | 06 Upgrade / Activate image | reloads; needs `allowActivation: true` |
| 10 | 09 Tasks / Get task, Get task tree | until finished |
| 11 | 07 Verify / Resync device, Get device version | |
| 12 | 08 Ports up / Build restore queue, Enable next port | re-send until queue empty |
| 13 | 10 Checks again | confirm ports are up |
| 14 | Clear `portSnapshot`, set both `allow*` switches back to false | |

Distribution is listed before the port shutdown on purpose: copying the image takes a while and has no impact,
so the ports only go down for the activation itself.

## Port handling

- Uses `PUT /dna/intent/api/v1/interface/{interfaceUuid}?deploymentMode=Deploy` with `{"adminStatus": "DOWN" | "UP"}`, one port per call.
- Only ports that were admin UP at snapshot time are shut, and only those are re-enabled.
- The snapshot is tied to the device; running "Snapshot ports" again keeps the original snapshot.
- The two "next port" requests loop by themselves in the Runner (run the folder); by hand, re-send until the "queue empty" test passes.

## Notes

- The "(new API)" requests use endpoints added during 2.3.7 (`/images`, `/networkDeviceImageUpdates`) and are informational.
- `networkDeviceImages/{id}/distribute|activate|update` are documented for 3.1.x and are not included.
- The intent API rate-limits; poll tasks every 15 to 30 s.
