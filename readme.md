# Last.fm Scrobbler for exteraGram

An automatic track scrobbler for Last.fm integrated with the exteraGram Telegram client.

## Overview

Last.fm Scrobbler for exteraGram is a tool that automatically submits tracks listened to within the [exteraGram](https://exteragram.app) Telegram client directly to your [Last.fm](https://www.last.fm) profile.

If you use Telegram as your primary music player and want to keep an accurate log of your listening history, this project provides a seamless, automated solution.

## Features

- **Automatic Scrobbling:** Tracks are automatically submitted to your Last.fm history as you listen in exteraGram.
- **Now Playing Status:** Real-time updates to your "Now Playing" status on Last.fm.
- **Lightweight:** Low resource consumption running in the background.
- **Easy Configuration:** Simple setup and authentication using the official Last.fm API.

## Requirements

Before you begin, ensure you have:

1. **exteraGram** installed on your device.
2. An active **Last.fm** account.
3. A Last.fm **API Key** and **Shared Secret** (obtainable via [Last.fm API Account Creation](https://www.last.fm/api/account/create)).

## Installation and Setup

### Step 1: Obtain Last.fm API Credentials

1. Visit the Last.fm API account creation page: https://www.last.fm/api/account/create
2. Fill in the required fields (Application name, Description, etc.).
3. Save the form and copy the generated credentials:
   - **API Key**
   - **Shared Secret**

### Step 2: Configure the Repository

1. Clone the repository:

   ```bash
   git clone https://github.com/Kruz-mm/Last.fm-scrobbler-for-exteraGram.git
   cd Last.fm-scrobbler-for-exteraGram
   ```

2. Set up the configuration file:
   - Rename `config.example.json` to `config.json` (or set the corresponding environment variables/settings).
   - Enter your `API Key` and `Shared Secret`.

3. Complete the authentication process by following the on-screen prompts.

## Usage

1. Launch the scrobbler or enable the module.
2. Open **exteraGram** and play any audio track from your chats or channels.
3. Check your [Last.fm](https://www.last.fm) profile; the track should appear under **"Scrobbling now"** or be added to your recent tracks after completion.

## FAQ

**Q: Why are some tracks not scrobbling?**  
A: Ensure that the audio files have proper ID3 tags (Artist and Track Title). Audio files without readable metadata may be skipped.

**Q: How much of a track needs to play before it is scrobbled?**  
A: By default, a scrobble is registered after 50% of the track duration has elapsed or after 4 minutes of listening, following Last.fm standard rules.

## Contributing

Contributions, issues, and feature requests are welcome!

1. Fork the project.
2. Create your feature branch (`git checkout -b feature/AmazingFeature`).
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`).
4. Push to the branch (`git push origin feature/AmazingFeature`).
5. Open a Pull Request.

## License

Distributed under the MIT License. See `LICENSE` for more information.