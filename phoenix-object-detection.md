## TL;DR

### Article Link
https://www.poeticoding.com/real-time-object-detection-with-phoenix-and-python/
### Author
Alvise Susmel 
## Key Takeaways
* How to interoperate between elixir and python's ML libraries
* How to calculate based on pixels using ports and a RTC webcamera

## Useful Code Snippets
```python
# detect.py
...
import time

start = time.time()
boxes, labels, _conf = cv.detect_common_objects(img, model="yolov3")
print("first detection: ", time.time() - start)

start = time.time()
boxes, labels, _conf = cv.detect_common_objects(img, model="yolov3")
print("second detection: ", time.time() - start)

```

```elixir
defmodule Yolo.Application do
  use Application

  def start(_type, _args) do
    children = [
      YoloWeb.Endpoint,
      
      # one worker named Yolo.Worker
      {Yolo.Worker, [name: Yolo.Worker]},
    ]

    opts = [strategy: :one_for_one, name: Yolo.Supervisor]
    Supervisor.start_link(children, opts)
  end

  ...
end
```
Upload
```elixir
def create(conn, %{"upload" => %Plug.Upload{}=upload}=_params) do
  data = File.read!(upload.path)
  detection = 
    Yolo.Worker.request_detection(Yolo.Worker, data) 
    |> Yolo.Worker.await()

  base64_image = base64_inline_image(data, upload.content_type)
  render(conn, "show.html", image: base64_image, detection: detection)
end

defp base64_inline_image(data, content_type) do
  image64 = Base.encode64(data)
  "data:#{content_type};base64, #{image64}"
end
```
Webcam JS
```javascript
// assets/js/webcam.js

//listen to "detected" events and calls draw_objects() for each event

channel.on("detected", draw_objects);

//our canvas element
let canvas = document.getElementById('objects');
let ctx = canvas.getContext('2d');
const boxColor = "blue";
//labels font size
const fontSize = 18;

function draw_objects(result) {
    
    let objects = result.objects;

    //clear the canvas from previews rendering
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.lineWidth = 4;
    ctx.font = `${fontSize}px Helvetica`;

    //for each detected object render label and box
    objects.forEach(function(obj) {
        let width = ctx.measureText(obj.label).width;
        
        // box
        ctx.strokeStyle = boxColor;
        ctx.strokeRect(obj.x, obj.y, obj.w, obj.h);

        // white label + background
        ctx.fillStyle = boxColor;
        ctx.fillRect(obj.x - 2, obj.y - fontSize, width + 10, fontSize);
        ctx.fillStyle = "white";
        ctx.fillText(obj.label, obj.x, obj.y - 2);
    });
}
const FPS = 1; // frames per second
let intervalID = null;

document.getElementById("start_stop")
        .addEventListener("click", function(){

  if(intervalID == null) {
      intervalID = setInterval(capture, 1000/FPS);
      this.textContent = "Stop";
  } else {
    clearInterval(intervalID);
    intervalID = null;
    this.textContent = "Start";
  }
});

export default socket
```
Detect.py
```
import cv2
import cvlib as cv

img = cv2.imread("dog.jpg")
boxes, labels, _conf = cv.detect_common_objects(img, model="yolov3")

print(labels, boxes)
```
## Useful Tools
* CV2
* RTC

## Comments/ Questions
