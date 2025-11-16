+++
title = "Archival: Lossless Audio Encoding"
date = 2025-10-17

emoji = "📀🎵🌟"
banner_c = "Local multimedia server music library banner graphic."

tags = ["archival", "encoding", "audio", "ffmpeg", "flac", "rsgain", "musicbrainz", "script"]
draft = false
+++

*TLDR: [The encode script](https://github.com/tbkfi/dotfile/blob/main/src/user/bin/encode-flac).*

Guess what. It's that time again when I get to clean up other people's subpar encoding jobs,
so I figured I'd go over my old tooling, refresh the code, and document the process while I'm at it.

Encoding lossless audio is much more simple than doing decent purpose specific lossy audio encoding -- not to mention video filtering and encoding,
which are often associated. So I'm a little confused as to why this seemingly simple simple task continues to give people a hard time.
The tools and libraries have existed (fundementally) unchanged for literally decades at this point, and they're well documented.
Regardless, the amount of inefficiently encoded and illegally tagged files in circulation continue to cause problems.

As an example, I like to store video game OSTs from games I've enjoyed on my media server.
Often many of the track variants are only available in the game files, and never make it to official releases.
Therefore the only official way to listen to the tracks again, aside from playing the game, is to extract the audio
from the game files, and to process them into an archivable format for safekeeping. Some generous people
spend a decent chunk of their time going through the trouble of doing this, but the output
is often weirdly formatted, uses illegal key and value fields in the embedded metadata, and lacks loudness
normalisation per track and album. This can lead to issues in playback, browsing, or inspection of the tracks.

![Media library with archived video game soundtracks](library-ost.jpg)

## Processing Steps

I'll demonstrate the full process step by step for a lossless demo file (*鳥の詩.flac*),
and at the end provide a full script that handles album directories for convenient batch processing.

*The demo file I'll use is actually properly encoded and tagged, I just grabbed something readily
available to illustrate the process!*

### PCM source and Data Integrity

If encoding to a lossless format like FLAC, you'd naturally want the original source file,
or something as close to it as possible. I start my encode process by restoring the processed
track into it's original WAV (PCM) format if it isn't already. This has two benefits, I discard all
the existing metadata that may be baked into the files and make sure all my files start from the same
"unaltered" point where only the signal is present.

Before I continue... Please don't encode lossy audio files (e.g. MP3, AAC, OPUS) into a lossless format,
it's stupid and misleading. You can't magically conjure up lost data by changing the codec, you'll only
bloat the output due to more inefficient encoding. There are technically some limited cases where doing
this makes sense, like re-mastering old audio tracks where the original masters are lost forever or otherwise
rendered inaccessible, but those people know what they're doing. If you have lossy files, be happy with them or source new rips.

Let's begin by making sure we have a clean and non-corrupted PCM file to work with:

```bash
#!/bin/bash

# Verify file integrity
# and meta fields, no output WAV.
flac --test "鳥の詩.flac"

# Verify file integrity,
# and write output WAV.
flac --decode "鳥の詩.flac"
```

Running flac with the decode flag, we get our desired output file `鳥の詩.wav` (if integrity is ok).
If the integrity checks fail, you'll have to rip or source a replacement file.

Sometimes sourcing a replacement is not possible if a track is extremely rare, and in those cases you can try to
use the `--force` option to output the decoded file even if the integrity checks fail. You'd
then want to closely inspect the audio file for defects and playback errors. Even if slightly damaged,
a lossless encode is valuable, specially in the absence of alternatives. Just don't forget to someway
mark that the file should be replaced in the future when/if possible due to corruption, because
after encoding it to FLAC the embedded checksum will indicate the file as OK, because the
corrupted signal is the new comparison point for the fresh FLAC checksum.

Let's quickly compare the processed source FLAC and its decoded WAV output by their properties:

```
$ mediainfo 鳥の詩.flac # source file

General
Complete name                            : 鳥の詩.flac
Format                                   : FLAC
Format/Info                              : Free Lossless Audio Codec
File size                                : 43.7 MiB
Duration                                 : 6 min 8 s
Overall bit rate mode                    : Variable
Overall bit rate                         : 997 kb/s
Album                                    : AIR ORIGINAL SOUNDTRACK
Album/Performer                          : Various Artists
Part                                     : 1/2
Part/Total                               : 2
Track name                               : Bird's Poem
Track name/Position                      : 03
Track name/Total                         : 26
Performer                                : Lia
Composer                                 : Shinji Orito
Genre                                    : Original Soundtrack
Recorded date                            : 2002
Cover                                    : Yes
Cover type                               : Cover (front)
Cover MIME                               : image/jpeg

Audio
Format                                   : FLAC
Format/Info                              : Free Lossless Audio Codec
Duration                                 : 6 min 8 s
Bit rate mode                            : Variable
Bit rate                                 : 977 kb/s
Channel(s)                               : 2 channels
Channel layout                           : L R
Sampling rate                            : 44.1 kHz
Bit depth                                : 16 bits
Compression mode                         : Lossless
Stream size                              : 42.9 MiB (98%)
Writing library                          : libFLAC 1.3.3 (2019-08-04)
MD5 of the unencoded content             : BA76F9EB07E6601A5960AFDE7D2A44B9
```

```
$ mediainfo 鳥の詩.wav # decoded WAV output

General
Complete name                            : 鳥の詩.wav
Format                                   : Wave
Format settings                          : PcmWaveformat
File size                                : 61.9 MiB
Duration                                 : 6 min 8 s
Overall bit rate mode                    : Constant
Overall bit rate                         : 1 411 kb/s

Audio
Format                                   : PCM
Format settings                          : Little / Signed
Codec ID                                 : 1
Duration                                 : 6 min 8 s
Bit rate mode                            : Constant
Bit rate                                 : 1 411.2 kb/s
Channel(s)                               : 2 channels
Sampling rate                            : 44.1 kHz
Bit depth                                : 16 bits
Stream size                              : 61.9 MiB (100%)
```

What's very evident is that we've lost our metadata (including embedded cover!) in the decode process.
This is desired, because we know that all our files will now be "clean" of any possibly
illegal meta fields that could cause problems due to unknown encoding, keys, or other format issues.
In other words, we have a clean base to work with when we later setup automatic tagging with
[Musicbrainz](https://musicbrainz.org/) and [Picard](https://picard.musicbrainz.org/).

### Encoding to FLAC

I know how allergic people are to looking up official documentation, so here's the
[link](https://xiph.org/flac/documentation_format_overview.html) to the FLAC format.

Encoding the PCM file to FLAC is made very easy, and the default settings in the encoder are
well regarded as sane; but, we may be able to do better than just sane. Or at least try to understand
more about how the encoder behaves and what parameters affect the output.
Proper control of encoding options, inclusion of a seektable, correctly formatted metadata blocks and,
if desired, reasonable padding provide a more resilient, usable, smaller, and possibly more performant output file that will
last into the future.

Generally speaking the options you need to care about in the encoding process are:
*the sample format*, *channel layout*, *encoder effort*, and *block size*. You could also
optimise other exposed encoder settings, but you'd be splitting hairs for such a small
amount of size savings that it's not worth it for general use and archival.

Below is an example of how the two main encoder settings (effort, blocksize) affects output filesizes
in the case of my demo file when encoding with 'flac' specifically:

```
=== EFFORT 8 ===
bs 4096: 44.933938 MB <- winner!
bs 2048: 44.974756 MB
bs 1024: 45.188216 MB
bs 512:  45.656543 MB
bs 256:  46.515364 MB

=== EFFORT 4 ===
bs 4096: 45.087671 MB
bs 2048: 45.124839 MB
bs 1024: 45.338003 MB
bs 512:  45.811137 MB
bs 256:  46.677598 MB

=== EFFORT 2 ===
bs 4096: 46.392430 MB
bs 2048: 46.400454 MB
bs 1024: 46.492873 MB
bs 512:  46.731520 MB
bs 256:  47.246093 MB
```

Quite predictably more "effort" results in a smaller output file. And in this case, of the tested
blocksizes 4096 came out ahead with best compression performance.

If you're working on a massive project with hundreds of thousands to millions of audio tracks, you'd
naturally want to optimise the hell out of each and every encode and only store the best output.
This would require automating the output filesize tests for the encode process, or curating a set
of library-specific encoding options that, in general, produce the best results for a genre of content.
While fun, definitely an overkill for my small library. I also want to stick to multiples of 4096
because my storage backend and network mounts use its multiples.

For my uses simply maxing out the effort and using 4096 as the blocksize is enough to get good
results semi-reliably (most of the time). And even if it misses a bit, it's not by much.

A general purpose encoding loop could look something like this:

```bash
#!/bin/bash

OUT_DIR="output"
mkdir "$OUT_DIR"

echo ">>> Encoding to FLAC"
shopt -s nocaseglob
for SRC in *.{wav,aiff,alac,flac}; do
	[[ -e "$SRC" ]] || continue
	FLAC="$OUT_DIR/${SRC%.*}.flac"
	T_START=$(date +%s)
	
	echo -n " * $SRC -> $FLAC ... "
	
	# FLAC already exists?
	if stat "$FLAC" &> /dev/null ; then
		echo "EXISTS"
		continue
	fi
	
	# Detect original sample format
	# see: https://ffmpeg.org/doxygen/7.0/group__lavu__sampfmts.html
	SAMPLE_FMT=$(ffprobe -v error -select_streams a:0 \
		-show_entries stream=sample_fmt \
		-of default=nokey=1:noprint_wrappers=1 "$SRC")
	
	PCM_FMT=""
	case "$SAMPLE_FMT" in
		s16) PCM_FMT=pcm_s16le ;;
		s24) PCM_FMT=pcm_s24le ;;
		s32) PCM_FMT=pcm_s32le ;;
		flt|fltp) PCM_FMT=pcm_s24le ;; # float-packed, float-planar
		dbl|dblp) PCM_FMT=pcm_s32le ;; # double-packed, double-planar
		*) PCM_FMT=pcm_s16le ;;
	esac
	
	# Detect channel layout and encode to FLAC
	ffmpeg -loglevel -8 -n -i "$SRC" \
		-af 'aformat=channel_layouts=mono|stereo|5.1|6.1|7.1' \
		-c:a "$PCM_FMT" -f wav - \
	| flac --best --blocksize=4096 --verify -o "$FLAC" - &>/dev/null
	ENC_RC=$?
	
	T_ELAPSED=$(( $(date +%s) - $T_START ))
	
	if (( $ENC_RC )); then
		echo "FAIL ($T_ELAPSED seconds)"
		continue
	else
		echo "OK ($T_ELAPSED seconds)"
	fi
	
	# Padding, Seektable
	metaflac --remove-all --dont-use-padding "$FLAC"
	metaflac --add-seekpoint=2s "$FLAC"
	metaflac --merge-padding "$FLAC"
    metaflac --sort-padding "$FLAC"
done
shopt -u nocaseglob
echo "DONE"
```

This script combines conversion to WAV before piping into `flac` for encoding, with intermediary steps to guess
proper channel layout and retain the original sample format via `ffprobe`.
The script therefore should also work for movie tracks with surround sound and other special releases
in addition to regular albums and EPs.

Let's see what the resulting FLAC file looks like after running the demo file through the previous script:

```
$ mediainfo output/鳥の詩.flac

General
Complete name                            : output/鳥の詩.flac
Format                                   : FLAC
Format/Info                              : Free Lossless Audio Codec
File size                                : 42.9 MiB
Duration                                 : 6 min 8 s
Overall bit rate mode                    : Variable
Overall bit rate                         : 977 kb/s

Audio
Format                                   : FLAC
Format/Info                              : Free Lossless Audio Codec
Duration                                 : 6 min 8 s
Bit rate mode                            : Variable
Bit rate                                 : 977 kb/s
Channel(s)                               : 2 channels
Channel layout                           : L R
Sampling rate                            : 44.1 kHz
Bit depth                                : 16 bits
Compression mode                         : Lossless
Stream size                              : 42.9 MiB (100%)
MD5 of the unencoded content             : BA76F9EB07E6601A5960AFDE7D2A44B9
```

```
$ metaflac --list output/鳥の詩.flac 

METADATA block #0
  type: 0 (STREAMINFO)
  is last: false
  length: 34
  minimum blocksize: 4096 samples
  maximum blocksize: 4096 samples
  minimum framesize: 16 bytes
  maximum framesize: 13608 bytes
  sample_rate: 44100 Hz
  channels: 2
  bits-per-sample: 16
  total samples: 16228800
  MD5 signature: ba76f9eb07e6601a5960afde7d2a44b9
METADATA block #1
  type: 3 (SEEKTABLE)
  is last: true
  length: 3312
  seek points: 184
    point 0: sample_number=0, stream_offset=0, frame_samples=4096
    point 1: sample_number=86016, stream_offset=145368, frame_samples=4096
    point 2: sample_number=176128, stream_offset=333243, frame_samples=4096
    point 3: sample_number=262144, stream_offset=510357, frame_samples=4096
    [ ... ]
    point 180: sample_number=15872000, stream_offset=44719829, frame_samples=4096
    point 181: sample_number=15962112, stream_offset=44833017, frame_samples=4096
    point 182: sample_number=16052224, stream_offset=44915332, frame_samples=4096
    point 183: sample_number=16138240, stream_offset=44933482, frame_samples=4096
```

The resulting FLAC file has the correct length, channel configuration, sampling rate, bit depth, and the FLAC metadata is clean with a seektable and no padding.

As an additional test I picked up a demo multi-channel fltp audio track from the Internet
to verify the script correctly preserves channel configuration and sample format.
Here are the original attributes of this other demo file (`dts-51.wav`):

*I couldn't find a MA (Master Audio) DTS track off the cuff, and didn't feel like ripping
one from a bluray for this, so I demoed on a lossy track. But the point still stands.*

```
$ mediainfo dts-51.wav # source file

General
Complete name                            : dts-51.wav
Format                                   : Wave
Format settings                          : PcmWaveformat
File size                                : 1.43 MiB
Duration                                 : 8 s 499 ms
Overall bit rate mode                    : Constant
Overall bit rate                         : 1 411 kb/s

Audio
Format                                   : DTS
Format/Info                              : Digital Theater Systems
Codec ID                                 : 1
Duration                                 : 8 s 499 ms
Bit rate mode                            : Constant
Bit rate                                 : 1 411.2 kb/s
Channel(s)                               : 6 channels
Channel layout                           : C L R Ls Rs LFE
Sampling rate                            : 44.1 kHz
Frame rate                               : 43.066 FPS (1024 SPF)
Bit depth                                : 24 bits
Compression mode                         : Lossy
Stream size                              : 1.43 MiB (100%)
```

```
$ mediainfo output/dts-51.flac # output file

General
Complete name                            : output/dts-51.flac
Format                                   : FLAC
Format/Info                              : Free Lossless Audio Codec
File size                                : 1.06 MiB
Duration                                 : 8 s 499 ms
Overall bit rate mode                    : Variable
Overall bit rate                         : 1 043 kb/s

Audio
Format                                   : FLAC
Format/Info                              : Free Lossless Audio Codec
Duration                                 : 8 s 499 ms
Bit rate mode                            : Variable
Bit rate                                 : 1 043 kb/s
Channel(s)                               : 6 channels
Channel layout                           : L R C LFE Ls Rs
Sampling rate                            : 44.1 kHz
Bit depth                                : 24 bits
Compression mode                         : Lossless
Stream size                              : 1.06 MiB (100%)
MD5 of the unencoded content             : F65874F2E65E51A59CCE5A0C833D3B97
```

We can again confirm that important characteristics like duration, sampling rate, bit depth,
and channel layout were preserved correctly during the encode. The printed layout ordering is
slightly different, but this is normal and doesn't affect the playback or the channel data itself.

### Loudness data (ReplayGain / EBU R 128)

The last "manual" step before mass tagging the albums/songs involves adding track and album specific loudness metadata.
This just makes listening to tracks and albums more pleasant through equalising the perceived loudness across
your audio library.

I like to use a program called `rsgain`. It supports both [ReplayGain](https://en.wikipedia.org/wiki/ReplayGain)
and [EBU R 128](https://en.wikipedia.org/wiki/EBU_R_128), and its easy to invoke.
Simply selecting the desired preset and pointing it at a directory (e.g. album) will make it
process each track separately and apply individual tags and album-specific loudness metadata
to all the tracks.

Below is an example on triggering rsgain for an album, and what the process output and resulting
metadata looks like:

```bash
#!/bin/bash

# Process all files in 'output/',
# and use the EBU R 128 preset.
rsgain easy -p ebur128 "output/"
```

```
$ rsgain easy -p ebur128 "World of Warcraft Taverns of Azeroth/"

[✔] Applying preset 'ebur128'...
[✔] Building directory tree...
[✔] Found 1 directory...
[✔] Scanning directory for files...
[✔] Scanning 'World of Warcraft Taverns of Azeroth/13. Thunderbrew.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/15. Smoke Lodge.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/19. Temple of the Moon.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/11. Bad Juju Bar & Grill.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/08. Salty Sailor.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/06. Shady Rest.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/01. Lion's Pride.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/16. Grunt's Place.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/07. Scarlet Raven.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/14. Hunter's Refuge.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/18. Bloodsail.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/03. Pig and Whistle.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/12. Slaughtered Lamb.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/05. Elders' Hearth.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/10. Spirit Stone.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/17. Tarren Mill.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/02. Stonefire.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/09. Deepwater.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]
[✔] Scanning 'World of Warcraft Taverns of Azeroth/04. Gallows' End.flac'
[✔] Container: raw FLAC [flac]
[✔] Stream #0: FLAC (Free Lossless Audio Codec), 16 bit, 44,100 Hz, 2 ch
 100% [============================================================================]

Track: World of Warcraft Taverns of Azeroth/13. Thunderbrew.flac
  Loudness:   -16.46 LUFS
  Peak:     0.994619 (-0.05 dB)
  Gain:        -6.54 dB 


Track: World of Warcraft Taverns of Azeroth/15. Smoke Lodge.flac
  Loudness:   -14.29 LUFS
  Peak:     1.035235 (0.30 dB)
  Gain:        -8.71 dB 


Track: World of Warcraft Taverns of Azeroth/19. Temple of the Moon.flac
  Loudness:   -15.51 LUFS
  Peak:     0.997152 (-0.02 dB)
  Gain:        -7.49 dB 


Track: World of Warcraft Taverns of Azeroth/11. Bad Juju Bar & Grill.flac
  Loudness:   -15.89 LUFS
  Peak:     1.077917 (0.65 dB)
  Gain:        -7.11 dB 


Track: World of Warcraft Taverns of Azeroth/08. Salty Sailor.flac
  Loudness:   -13.26 LUFS
  Peak:     1.272733 (2.09 dB)
  Gain:        -9.74 dB 


Track: World of Warcraft Taverns of Azeroth/06. Shady Rest.flac
  Loudness:   -16.30 LUFS
  Peak:     0.891467 (-1.00 dB)
  Gain:        -6.70 dB 


Track: World of Warcraft Taverns of Azeroth/01. Lion's Pride.flac
  Loudness:   -13.95 LUFS
  Peak:     1.020671 (0.18 dB)
  Gain:        -9.05 dB 


Track: World of Warcraft Taverns of Azeroth/16. Grunt's Place.flac
  Loudness:   -16.83 LUFS
  Peak:     1.005116 (0.04 dB)
  Gain:        -6.17 dB 


Track: World of Warcraft Taverns of Azeroth/07. Scarlet Raven.flac
  Loudness:   -16.06 LUFS
  Peak:     0.994162 (-0.05 dB)
  Gain:        -6.94 dB 


Track: World of Warcraft Taverns of Azeroth/14. Hunter's Refuge.flac
  Loudness:   -16.16 LUFS
  Peak:     1.020689 (0.18 dB)
  Gain:        -6.84 dB 


Track: World of Warcraft Taverns of Azeroth/18. Bloodsail.flac
  Loudness:   -13.63 LUFS
  Peak:     1.022600 (0.19 dB)
  Gain:        -9.37 dB 


Track: World of Warcraft Taverns of Azeroth/03. Pig and Whistle.flac
  Loudness:   -13.87 LUFS
  Peak:     1.009166 (0.08 dB)
  Gain:        -9.13 dB 


Track: World of Warcraft Taverns of Azeroth/12. Slaughtered Lamb.flac
  Loudness:   -14.12 LUFS
  Peak:     1.030378 (0.26 dB)
  Gain:        -8.88 dB 


Track: World of Warcraft Taverns of Azeroth/05. Elders' Hearth.flac
  Loudness:   -16.26 LUFS
  Peak:     1.000769 (0.01 dB)
  Gain:        -6.74 dB 


Track: World of Warcraft Taverns of Azeroth/10. Spirit Stone.flac
  Loudness:   -16.07 LUFS
  Peak:     1.040404 (0.34 dB)
  Gain:        -6.93 dB 


Track: World of Warcraft Taverns of Azeroth/17. Tarren Mill.flac
  Loudness:   -15.05 LUFS
  Peak:     0.992078 (-0.07 dB)
  Gain:        -7.95 dB 


Track: World of Warcraft Taverns of Azeroth/02. Stonefire.flac
  Loudness:   -13.76 LUFS
  Peak:     1.014163 (0.12 dB)
  Gain:        -9.24 dB 


Track: World of Warcraft Taverns of Azeroth/09. Deepwater.flac
  Loudness:   -14.87 LUFS
  Peak:     1.037309 (0.32 dB)
  Gain:        -8.13 dB 


Track: World of Warcraft Taverns of Azeroth/04. Gallows' End.flac
  Loudness:   -15.45 LUFS
  Peak:     0.821682 (-1.71 dB)
  Gain:        -7.55 dB 

Album:
  Loudness:   -15.02 LUFS
  Peak:     1.272733 (2.09 dB)
  Gain:        -7.98 dB 


Scanning Complete
Time Elapsed:      00:00:11
Files Scanned:     19
Clip Adjustments:  0 (0.0% of files)
Average Gain:      -7.85 dB
Average Peak:      1.014648 (0.13 dB)
Negative Gains:    19 (100.0% of files)
Positive Gains:    0 (0.0% of files)
```

```
$ mediainfo "06. Shady Rest.flac"

General
Complete name                            : 06. Shady Rest.flac
Format                                   : FLAC
Format/Info                              : Free Lossless Audio Codec
File size                                : 14.1 MiB
Duration                                 : 2 min 49 s
Overall bit rate mode                    : Variable
Overall bit rate                         : 696 kb/s
Album replay gain                        : -7.98 dB
Album replay gain peak                   : 1.272733

Audio
Format                                   : FLAC
Format/Info                              : Free Lossless Audio Codec
Duration                                 : 2 min 49 s
Bit rate mode                            : Variable
Bit rate                                 : 696 kb/s
Channel(s)                               : 2 channels
Channel layout                           : L R
Sampling rate                            : 44.1 kHz
Bit depth                                : 16 bits
Compression mode                         : Lossless
Replay gain                              : -6.70 dB
Replay gain peak                         : 0.891467
Stream size                              : 14.0 MiB (100%)
MD5 of the unencoded content             : 83852F0A685AE6E79729C63DC9B0E52B
```

With that your albums should be equalised for loudness quite nicely. Yay for no more blown
ear drums when the songs change, or albums switch.

### Music Metadata via MusicBrainz

Tagging the music is quite an endeavour on its own, even with the help of databases like
MusicBrainz and tools like Picard.

I would recommend batch processing multiple albums into an ingest directory, and then using
Picard's graphical UI to apply any changes in bulk once you've accumulated enough new albums.
Always giving a visual pat-down of any new changes is a good idea, the matching isn't perfect.

You can tune the settings in-application easily, and dictate what kind of album metadata and art should
be applied from the public database to your files. But don't embed images into the files directly!
A single high-quality cover art image at the root of the album titled `cover.jpg` is the most efficint way
of handling cover art. There's no sense in duplicating a high-quality 5MB cover art into every song in the album,
when one separate image is enough.

Also, If you see any missing data in the public database (missing albums, song metadata, etc.), don't
be afraid to contribute to the database. The account is free to create and it's better there is
some information than none available for a release, that way people know a release at least exists!

If you're entirely new to the site, try to apply the monkey-see monkey-do approach and see how other people have
added or edited releases in the past (specially the trusted contributors!). That'll get you close enough to get started,
the rest of the style guides you can ape directly from the official [MusicBrainz contributing guidelines](https://musicbrainz.org/doc/How_to_Contribute).

## Automated script

The script below will accept a file as its first argument and encode it as previously
discussed into a lossless FLAC. An optional second argument can be given to specify an output
directory. By default the script writes the encoded file next to the original with the original name
and an `_ENCODE` suffix when a custom DST isn't given.

```bash
#!/bin/bash

encode_track_to_flac() {
	T_START=$(date +%s)
	
	# Error codes
	RC_ERR_SOURCE=1
	RC_ERR_OUTPUT=2
	RC_ERR_ENCODE=3
	
	# Arguments
	SRC="$1";
	DST="$2";
	
	echo -n "[EncodeFlac]: "
	
	# SRC: was given?
	if [[ -z "$SRC" ]]; then
		echo "NO SOURCE PROVIDED!"
		return $RC_ERR_SOURCE;
	fi
	
	# SRC: exists?
	if ! stat "$SRC" &>/dev/null; then
		echo "DOESN'T EXIST!"
		return $RC_ERR_SOURCE
	fi
	
	# SRC: read access?
	if ! [[ -r "$SRC" ]]; then
		echo "NOT READABLE!"
		return $RC_ERR_SOURCE
	fi

	echo "'$SRC'"
	TRACK_NAME="$(basename "$SRC")"
	TRACK_PATH="$(dirname "$(readlink -f "$SRC")")"
	
	# Custom Destination (optional)
	if [[ -n "$DST" ]]; then
		DST="$(realpath "$DST")"
		mkdir -p "$DST"
		
		[[ -d "$DST" && -w "$DST" ]] || return $RC_ERR_OUTPUT
		FLAC="$DST/${TRACK_NAME%.*}.flac"
	else
		FLAC="$TRACK_PATH/${TRACK_NAME%.*}_ENCODE.flac"
	fi
	echo " -> DST: '$FLAC'"
	echo -n " -> STATUS: "
	
	# Abort if FLAC exists
	if stat "$FLAC" &> /dev/null ; then
		echo -e "ENCODE EXISTS\n"
		return $RC_ERR_OUTPUT
	fi
	
	# Detect original sample format
	# see: https://ffmpeg.org/doxygen/7.0/group__lavu__sampfmts.html
	SAMPLE_FMT=$(ffprobe -v error -select_streams a:0 \
		-show_entries stream=sample_fmt \
		-of default=nokey=1:noprint_wrappers=1 "$SRC")
	
	PCM_FMT=""
	case "$SAMPLE_FMT" in
		s16) PCM_FMT=pcm_s16le ;;
		s24) PCM_FMT=pcm_s24le ;;
		s32) PCM_FMT=pcm_s32le ;;
		flt|fltp) PCM_FMT=pcm_s24le ;; # float-packed, float-planar
		dbl|dblp) PCM_FMT=pcm_s32le ;; # double-packed, double-planar
		*) PCM_FMT=pcm_s16le ;;
	esac
	
	# SRC -> PCM -> FLAC
	ffmpeg -loglevel -8 -n -i "$SRC" \
		-af 'aformat=channel_layouts=mono|stereo|5.1|6.1|7.1' \
		-c:a "$PCM_FMT" -f wav - \
	| flac --best --blocksize=4096 --verify -o "$FLAC" - &>/dev/null
	ENC_RC=$?
	
	T_ELAPSED=$(( $(date +%s) - $T_START ))
	
	if (( $ENC_RC )); then
		echo -e "FAIL ($T_ELAPSED seconds)\n"
		return $RC_ERR_ENCODE
	else
		echo -e "OK ($T_ELAPSED seconds)\n"
	fi
	
	# Padding, Seektable
	metaflac --remove-all --dont-use-padding "$FLAC"
	metaflac --add-seekpoint=2s "$FLAC"
	metaflac --merge-padding "$FLAC"
	metaflac --sort-padding "$FLAC"
}

if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then
	encode_track_to_flac "$1" "$2"
fi
```

You're free to choose how to invoke the script, but a simple batch example could be achieved
with either `find` or `fd` (e.g. fd-find) to run the script for each extension matched file, like so:

```bash
# Find all 'flac' and 'wav' files,
# and pass the filepath as first argument,
# and output to current dir '.' (second argument).
$ fd -e flac -e wav -x ./encode_track.sh "{}" .

[EncodeFlac]: './14. Twisted Force.flac'
 -> DST: '/home/user/local/music/enc/14. Twisted Force.flac'
 -> STATUS: ENCODE EXISTS

[ ... ]

[EncodeFlac]: './Baldur's Gate 3 Original Soundtrack/43. The Power (Credits Song).flac'
 -> DST: '/home/user/local/music/enc/43. The Power (Credits Song).flac'
 -> STATUS: ENCODE EXISTS

[EncodeFlac]: './Baldur's Gate 3 Original Soundtrack/08. Harpy Song.flac'
 -> DST: '/home/user/local/music/enc/08. Harpy Song.flac'
 -> STATUS: OK (1 seconds)

[EncodeFlac]: './Baldur's Gate 3 Original Soundtrack/04. Who Are You.flac'
 -> DST: '/home/user/local/music/enc/04. Who Are You.flac'
 -> STATUS: OK (2 seconds)

[EncodeFlac]: './Baldur's Gate 3 Original Soundtrack/01. Main Theme Part I.flac'
 -> DST: '/home/user/local/music/enc/01. Main Theme Part I.flac'
 -> STATUS: OK (2 seconds)

[ ... ]

```

The script will output either "OK (time elapsed)" or "FAIL (time elapsed)", depending on
if the encode process returns anything other than 0 (success). In the event that the DST FLAC
exists already, the status will say "ENCODE EXISTS" like in the above output.

That's all.
