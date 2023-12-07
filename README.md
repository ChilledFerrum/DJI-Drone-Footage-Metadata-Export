# DJI drone footage metadata export tool

> Exports the metadata recorded in DJI drone footage

<details>
<summary>Code Explanation</summary>

<br/>

**imports**
```python
import subprocess as sp
from moviepy.editor import VideoFileClip
import json
import os
import argparse
```
<br/>


**Available formats**
In most cases the expected video file format is ".MP4"
```py
allowed_extensions = ['.mp4', '.mov', '.avi', '.mpg', '.mkv']
```
<br/>

**Arguments**
Arguments to allow the user to execute the code with the desired parameters
```py
def parse_opt():
    parser = argparse.ArgumentParser()
    parser.add_argument('--input', '-i', type=str, default="dji_videos/", nargs='+', help="Input Directory containing the DJI videos")
    parser.add_argument('--export', '-e', type=str, default="src_vid_info/", help="Export Directory to generate the metadata jsons")
    parser.add_argument('--interval', '--int','-int', nargs='+', type=int, default=3, help='For 3 Seconds Delay --interval 3')
    parser.add_argument('--export-query', '-eq', type=str, default="data/data.json", help="Export Query Path Directory e.g. data/data.json")
    parser.add_argument('--vid-format', type=str, default=".mp4", help="Format of your videos ('mp4', 'mov', 'avi', etc...)")
    opt = parser.parse_args()
    return opt
```
<br/>

**Video Processing & exporting**
Processing and exporting of each individual video file located in the input directory folder containing DJI video files. ('.MP4')
```py
def export_info(args):
    vids_par_path = args.input
    export_path = args.export
    export_query_path = args.export_query
    src_vids_path = [file for file in os.listdir(vids_par_path) if any(file.lower().endswith(ext) for ext in allowed_extensions)] # Filter anything 


    insert_query_dct = {"vid_count:": len(src_vids_path),
                    "Videos": []}
    for vid_name in src_vids_path:
        vid_path = vids_par_path + vid_name  # Parent + Child path
        export_filename = vid_name.split(".")[0]
        print(export_filename, " - ")
        information = extract_metadata(args, vid_path, vid_name)
        print(export_filename, " - COMPLETED")

        with open(f"{export_path}{export_filename}.json", 'w') as f:
            json.dump(information, f, indent=1)
        insert_query_dct["Videos"].append(information)
        print(f"DJI metadata for video (\033[1m{vid_name}\033[0m) generated at (\033[1m{vid_path}\033[0m)")

    with open(export_query_path, 'w') as f:
        json.dump(insert_query_dct, f, indent=1)
    print(f"\nDJI Video metadata json file generated at {export_query_path}")

```
 
**Metadata extraction**
Extraction of the metadata information, recorded in the DJI drone footage. Works by interacting with each second of the subtitles of the video
```py
def extract_metadata(args, video_path, vid_name):
    intervals = args.interval
    video_clip = VideoFileClip(video_path)
    duration = video_clip.duration
    resolution = video_clip.size

    information = {"filename": video_path,
                   "duration": duration,
                   "resolution": resolution,
                   "frame_info": []}

    out = sp.run(['ffmpeg', '-i', video_path, '-map', 's:0', '-f', 'webvtt', '-'],
                 stdout=sp.PIPE, stderr=sp.PIPE, universal_newlines=True)
    results = out.stdout.splitlines()
    frame_information = {}
    
    three_second_rule = 0
    second = ""
    for line in results:
        if "-->" in line:
            current_dur = line.split("-->")[0]
            second = current_dur
            three_second_rule -= 1

        if "F/5" in line:
            metadata = line.split(',')
            gps = format_gps(metadata[4:7])
            frame_information["second"] = second
            frame_information["altitude"] = metadata[8].split(" ")[2]  # ALTITUDE
            frame_information["gps"] = gps  # GPS INFO
            frame_information["dst_from_HQ"] = metadata[7].split(" ")[2]  # DISTANCE FROM LAUNCH
            frame_information["hs"] = metadata[9]  # HORIZONTAL SPEED
            frame_information["vs"] = metadata[10]

            if three_second_rule <= 0:
                information["frame_info"].append(frame_information)
                three_second_rule = intervals
            frame_information = {}
            
    return information
```
</details>


#### Code Execution
```bash
pip install -r requirements
python export_video_metadata.py
```