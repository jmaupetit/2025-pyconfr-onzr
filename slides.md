---
theme: default
background: #222
title: "Onzr : l'histoire du CLI Deezer un peu en retard"
info: |
  Parce que Deezer ne fournit pas de client Linux officiel, parce que leur
  player web utilise une quantité absurde de RAM, parce que l'informatique c'est
  fun, parce qu'apprendre des nouveaux trucs c'est stimulant, parce que tout
  faire depuis son terminal c'est cool, je me suis lancé dans l'écriture d'un
  outil en ligne de commande pour streamer ma musique.

  Je me propose de vous conter l'histoire de ce side-project
  [open-source](https://github.com/jmaupetit/onzr).

class: text-center
drawings:
  persist: false
transition: slide-left
mdc: false
# duration of the presentation
duration: 20min
---

# `onzr`

## L'histoire du CLI Deezer un peu en retard…

• PyConFR 💜 Lyon 💜 2 novembre 2025 •

[Julien Maupetit](https://julien.maupetit.me)

---
layout: image-right 
image: /images/me.jpg
---

# /me

- Pythonista since `2.2`
- Open-`(Source|Data|Science)` enthousiast
- Code craftsman
- Wanabee musician
- UNIX / CLI lover

---
layout: center
---

## Once upon a time…

---
layout: two-cols-header
---

# Jan 27, 2025 · Leaving Spotify

::left::

[
  ![Spotify toot](/images/spotify-toot.jpg)
](https://mamot.fr/@jmaupetit/113899073423698676)

::right::

[
    ![Spotify shame](/images/spotify-shame.jpg)
](https://www.lesinrocks.com/musique/investiture-de-donald-trump-spotify-a-fait-don-de-150-000-dollars-pour-la-ceremonie-649529-23-01-2025/)

<style>
.two-cols-header {
  column-gap: 40px;
}
</style>

---
layout: two-cols-header
---

# Feb 4, 2025 · Hello Deezer

::left::

[
  ![Deezer toot 1](/images/deezer-toot-01.jpg)
](https://mamot.fr/@jmaupetit/113944690650791722)

[
  ![Deezer toot 2](/images/deezer-toot-02.jpg)
](https://mamot.fr/@jmaupetit/113944702684503435)

::right::

[
  ![Deezer toot 3](/images/deezer-toot-03.jpg)
](https://mamot.fr/@jmaupetit/113944739914392009)

<style>
.two-cols-header {
  column-gap: 40px;
}
</style>

---
layout: image-right
image: /images/the-office-sad.gif
---

# Deezer + GNU/Linux

- No official desktop client
- Only the web player that… mostly work. Most of the time.

![Deezer error](/images/deezer-error.jpg)

- By the end of the day, the player tab often uses more than 2Gb of RAM.

---
layout: image-right
image: /images/the-office-plan.gif
---

# What if…

… I write my own tool to listen to my music?

<br/>

I 💜 Command Line Interfaces.

I know a bit of Python.

It's time to experiment.

---
layout: center 
---

# How do we play Deezer music with Python?

---

# Streaming Deezer tracks

There are many Deezer API clients for Python.

I use [`deezer-py`](https://pypi.org/project/deezer-py/) (discovered via [StreamRip](https://pypi.org/project/streamrip/)) 🤫

<br/>

```python {*|6-9|1,2,11|1,2,11-15,17}{lines:true,startLine:1}
chunk_sep = 2048
chunk_size = 3 * chunk_sep
self.streamed = 0
self.status = TrackStatus.IDLE

# The URL has been generated given a quality and a track token using the API client
with self.session.get(url, stream=True) as r:
    r.raise_for_status()
    filesize = int(r.headers.get("Content-Length", 0))

    for chunk in r.iter_content(chunk_size):
        if len(chunk) > chunk_sep:
            dchunk = self._decrypt(chunk[:chunk_sep]) + chunk[chunk_sep:]
        else:
            dchunk = chunk
        self.streamed += chunk_size
        yield dchunk
```

---
layout: center 
---

# How do we play sound with Python?

---
layout: image-right
image: /images/the-office-music.gif
---

# Play sound 

There are many libraries available.

- [Playsound](https://pypi.org/project/playsound/)
- [Simpleaudio](https://pypi.org/project/simpleaudio/)
- [Pydub](https://pypi.org/project/pydub/)

But some requires `ffmpeg` for transcoding or strongly depends on the OS sound
server/backend.

I was stuck, until I found…

---
layout: image-right 
image: /images/the-office-wow.gif
---

# VLC bindings for python

It runs everywhere.

Run and control a headless VLC instance using Python.

```python 
import vlc

player = vlc.MediaPlayer('file:///tmp/foo.flac')
player.play()
```

---
layout: statement
---

# Quick summary

For those who were sleeping.

We can decrypt Deezer audio streams.

We can play those streams everywhere.

But. 

❌ We don't want to download/store files and then play them, 

✅ We want to play streams on-the-fly.

---

# UDP Multicasting

Epic fail.

```python {*}{maxHeight:'400px'}
class Track:
    """A Deezer track."""
    
    # [...]

    def _allocate_content(self) -> None:
        """Allocate memory where we will read/write track bytes."""
        self.content = bytearray(self.filesize)
        self._content_mv = memoryview(self.content)

    def fetch(self):
        """Fetch track in-memory.

        buffer_size (int): the buffer size (defaults to 5 seconds for a 128kbs file)
        """
        logger.debug(f"Start fetching track with {self.buffer_size=}")
        chunk_sep = 2048
        chunk_size = 3 * chunk_sep
        self.fetched = 0
        self.status = TrackStatus.IDLE
        self._allocate_content()

        with self.session.get(self.url, stream=True) as r:
            r.raise_for_status()
            filesize = int(r.headers.get("Content-Length", 0))
            logger.debug(f"Track size: {filesize} ({self.filesize})")
            self.status = TrackStatus.FETCHING

            for chunk in r.iter_content(chunk_size):
                if len(chunk) > chunk_sep:
                    dchunk = self._decrypt(chunk[:chunk_sep]) + chunk[chunk_sep:]
                else:
                    dchunk = chunk
                self._content_mv[self.fetched : self.fetched + chunk_size] = dchunk
                self.fetched += chunk_size

                if (
                    self.fetched >= self.buffer_size
                    and self.status < TrackStatus.PLAYABLE
                ):
                    logger.debug("Buffering ok")
                    self.status = TrackStatus.PLAYABLE

        # We are done here
        self.status = TrackStatus.FETCHED
        logger.debug("Track fetched")

    def cast(self, socket, chunk_size: int = 1024):
        """Cast the track via UDP using given socket."""
        multicast_group = tuple(settings.MULTICAST_GROUP)
        logger.debug(
            (
                f"Casting from position {self.streamed} with {chunk_size=} "
                f"using socket {socket} "
                f"({multicast_group=})"
            )
        )

        if self.status < TrackStatus.FETCHING:
            logger.debug("Will start fetching content in a new thead…")
            thread = Thread(target=self.fetch)
            thread.start()

        # Wait for the track to be playable
        while self.status < TrackStatus.PLAYABLE:
            sleep(0.01)

        # Sleep time while playing
        wait = 1.0 / (self.bitrate / chunk_size)
        logger.debug(f"Wait time: {wait}s ({self.bitrate=})")

        slow_connection: bool = False
        for start in range(self.streamed, self.filesize, chunk_size):
            # Track has been paused, wait for resume
            while self.paused:
                sleep(0.001)

            # We have buffering issues
            while (self.fetched - start) < self.buffer_size and start < (
                self.filesize - self.buffer_size
            ):
                if not slow_connection:
                    logger.warning(
                        "Slow connection, filling the buffer "
                        f"{self.fetched - self.streamed} < {self.buffer_size}"
                    )
                    logger.debug(
                        f"{start=} | {self.filesize=} | {self.fetched=} | "
                        f"{self.streamed=} | {chunk_size=}"
                    )
                slow_connection = True
                sleep(0.05)
            slow_connection = False

            socket.sendto(self._content_mv[start : start + chunk_size], multicast_group)
            self.streamed += chunk_size
            sleep(wait)

```

---

# FastAPI HTTP server 

I can haz API 🤩

```python {*}{maxHeight:'400px'}
@app.get(settings.TRACK_STREAM_ENDPOINT)
async def stream_track(
    onzr: Annotated[Onzr, Depends(get_onzr)],
    rank: Annotated[int, Path(title="Track queue rank")],
) -> StreamingResponse:
    """Stream Deezer track given its identifer."""
    if onzr.queue.is_empty:
        raise HTTPException(
            status_code=status.HTTP_204_NO_CONTENT, detail="Queue is empty."
        )
    if rank >= len(onzr.queue):
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, detail="Track rank out of range."
        )
    onzr.queue.playing = rank
    track = onzr.queue[rank]
    # Refresh track token in case it expired
    track.refresh()
    quality = track.query_quality(settings.QUALITY)
    return StreamingResponse(track.stream(quality), media_type=quality.media_type)
```

---
layout: center 
---

# Demo 

---
layout: center 
---

# One last word…

---
layout: statement
---

## Don't forget 

# <v-click>coding</v-click> 
# <v-click>experimenting</v-click> 
# <v-click>learning</v-click> 

## is fun.

---
layout: center
---

💜 Thank you 💜

---

# Credits

- GIFs from "The Office" comes from their [Giphy channel](https://giphy.com/theoffice)
