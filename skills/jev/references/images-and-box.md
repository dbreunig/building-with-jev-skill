# Images and Box

This reference covers sending images to Jev and asking Box questions, which locate a target in an image.

## Status and setup

TypeSafe calls images and Box "dreamer-ready". Use them for prototypes. The request format, the answers, and the limits can change between releases.

Send the `TypeSafe-Danger-Zone-Images: true` header with every request that has an image or a Box question. A request without the header returns `400`. In Python, pass `headers=` to the client or `extra_headers=` to `system_one`.

## Example

```python
from pathlib import Path
from typesafe_sdk import Box, Image, Noul, TypeSafeClient

with TypeSafeClient(model="jev-1.14.0", headers={"TypeSafe-Danger-Zone-Images": "true"}) as client:
    response = client.system_one(
        state={"photo": Image(Path("kitchen.jpg"))},
        questions={
            "mug_visible": Noul(instructions="Is there a coffee mug in `photo`?"),
            "mug": Box(instructions="the coffee mug in `photo`"),
        },
    )

if response.nouls["mug_visible"].noul > 0.7:
    edges = response.boxes["mug"].box  # top, bottom, left, right, each from 0 to 1
```

## Placing images

- An image can go in the `state`, in `instructions`, or in `criteria`. Every question type accepts images.
- Put an image that several questions use in the `state` once, and refer to it by path. Each place a reference appears adds another copy of the image to the tokens.
- Put a known picture in `instructions` to compare against, e.g., "Is the person in `door` the same person as in this badge photo: {badge}?"
- Define Choice options, Score levels, or Noul sides with example pictures in `criteria`. Option names stay text.
- Send a short frame history in the state to ask what changed.
- The SDK creates the asset references. Over raw HTTP, list each image once in the top-level `assets` map, and use a fresh UUID as its reference. Every asset must be referenced.

## Box questions

- Box needs `jev-1.14` or later. Its instructions are a noun phrase that names the target and the image, e.g., "the coffee mug in `photo`". Box takes no criteria.
- A Box always returns a box, even when the target is absent. Ask a presence Noul in the same request and ignore the box when the Noul is low.
- One Box finds one target. Ask one Box per target in the same request.
- Name the target you want. Position words such as "leftmost" can land two boxes on the same object. When code finds two boxes that overlap heavily, keep one.
- Edges are fractions of the image size. Multiply by the width or height in pixels to crop.
- No recommended confidence threshold exists yet. When confidence is low, pad the box before cropping, or crop to the region and ask again on the crop.
- To find a small target in a large image, ask a Choice with tiles of the image as options first, then ask the Box on the chosen tile.

## Reading and judging

- Read values from the image in one request. Judge those readings as text in a second request. A single question that does both hides which step failed.
- To check another tool's reading, e.g., OCR output, ask Jev whether the value matches the full image and whether it matches a crop of its box. Flag the value when either check is low.
- Add `unreadable` and `other` options to a Choice that reads from a scan.

## Preparing images

- Send PNG or JPEG still images. Convert GIF, WebP, and HEIC first.
- Each side must be 280 to 1568 pixels. Scale the longest side to 1568 or less. The API rejects images outside the limits and does not resize them.
- The API strips EXIF data and ignores EXIF orientation. Rotate the pixels upright before sending, e.g., with `ImageOps.exif_transpose`.
- Each image file can be at most 16 MiB. All images in a request share 32 MiB. Images count as input tokens at the normal price.
