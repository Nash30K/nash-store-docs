# Guides

Complete walkthroughs for the tasks a server owner actually performs, from an untouched
install to a phone that matches the server.

Each guide follows the same shape: the goal in one sentence, the prerequisites, numbered
steps with the exact code to paste, and a verification section that tells you how to know it
worked.

## Before opening the server

- [Set up image hosting](setup-image-hosting.md): **mandatory**. Without it the camera, the
  gallery and every profile picture are unable to save anything.
- [Choose which apps are installed](customize-apps.md): enable, rename, preinstall or hide
  any of the shipped applications.
- [Add locations to the Maps app](add-map-locations.md): replace the sample points with the
  landmarks of your own map.

## Content and gameplay

- [Fill the music library](add-music.md): the library ships empty on purpose.
- [Wire your jobs into the Services app](setup-services.md): police, EMS, mechanic, taxi, or
  anything else your framework calls a job.
- [Add a language](add-language.md): one JSON file, one config line, no rebuild.

## Going further

- [Add a third-party application](add-custom-app.md): ship your own app from your own
  resource, or port one written for another phone.
- [Video calls and InstaPic live streams](setup-video-calls.md): what travels through your
  server, what travels player to player, and when you need a TURN server.

## Related

- [Configuration Files](../config/README.md): reference for every key in `config/`
- [Developer API](../developer-api/README.md): exports, events and callbacks
- [Common Errors](../common-errors.md): symptom, cause, fix
