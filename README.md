# imgproxy.rb

<img align="right" width="200" height="200" title="imgproxy logo"
     src="https://cdn.rawgit.com/DarthSim/imgproxy/master/logo.svg">

[![CircleCI branch](https://img.shields.io/circleci/project/github/imgproxy/imgproxy.rb/master.svg?style=for-the-badge)](https://circleci.com/gh/imgproxy/imgproxy.rb) [![Gem](https://img.shields.io/gem/v/imgproxy.svg?style=for-the-badge)](https://rubygems.org/gems/imgproxy) [![rubydoc.org](https://img.shields.io/badge/rubydoc-reference-blue.svg?style=for-the-badge)](https://www.rubydoc.info/gems/imgproxy)

**[imgproxy](https://github.com/imgproxy/imgproxy)** is a fast and secure standalone server for resizing and converting remote images. The main principles of imgproxy are simplicity, speed, and security. It is a Go application, ready to be installed and used in any Unix environment—also ready to be containerized using Docker.

imgproxy can be used to provide a fast and secure way to replace all the image resizing code of your web application (like calling ImageMagick or GraphicsMagick, or using libraries), while also being able to resize everything on the fly, fast and easy. imgproxy is also indispensable when handling lots of image resizing, especially when images come from a remote source.

**imgproxy.rb** is a Ruby Gem for imgproxy that is framework-agnostic, but includes proper support for Ruby on Rails' most popular image attachment options.

imgproxy.rb provides easy configuration and URL generation as well as plug&play support for [Active Storage](https://edgeguides.rubyonrails.org/active_storage_overview.html) and [Shrine](https://github.com/shrinerb/shrine).

<a href="https://evilmartians.com/?utm_source=imgproxy.rb">
<img src="https://evilmartians.com/badges/sponsored-by-evil-martians.svg" alt="Sponsored by Evil Martians" width="236" height="54">
</a>

## Installation

Add this to your `Gemfile`:

```ruby
gem "imgproxy"
```

or install system-wide:

```
gem install imgproxy
```

## Configuration

Next, some basic configuration. We will use a Ruby on Rails application as an example; place the following code to `config/initializers/imgproxy.rb`:

```ruby
# config/initializers/imgproxy.rb

Imgproxy.configure do |config|
  # imgproxy endpoint
  #
  # Full URL to where your imgproxy lives.
  config.endpoint = "http://imgproxy.example.com"

  # Next, you have to provide your signature key and salt.
  # If unsure, check out https://github.com/imgproxy/imgproxy/blob/master/docs/configuration.md first.

  # Hex-encoded signature key
  config.hex_key = "your_key"
  # Hex-encoded signature salt
  config.hex_salt = "your_salt"

  # Base64 encode all URLs
  # config.base64_encode_urls = true
end
```

## Usage

### Using with Active Storage

imgproxy.rb comes with built-in Active Storage support. To enable it, modify your initializer at `config/initializers/imgproxy.rb`:

```ruby
# config/initializers/imgproxy.rb

Imgproxy.extend_active_storage!
```

Now, to add imgproxy processing to your image attachments, just use the `imgproxy_url` method:

```ruby
user.avatar.imgproxy_url(width: 250, height: 250)
```

will give you an URL to your user's avatar, resized to 250x250px on the fly.

If you have configured both your imgproxy server and Active Storage to work with Amazon S3, you may want to use the short `s3://...` source URL scheme instead of the long one generated by Rails:

```ruby
# config/initializers/imgproxy.rb

Imgproxy.extend_active_storage!(use_s3: true)
```

You can do the same if you are using Google Cloud Storage:

```ruby
# config/initializers/imgproxy.rb

Imgproxy.extend_active_storage!(use_gcs: true, gcs_bucket: "my_bucket")
```

**Note** that you need to explicitly provide GCS bucket name since Active Storage "hides" the GCS config.

### Using with Shrine

You can also use imgproxy.rb's built-in [Shrine](https://github.com/shrinerb/shrine) support. To enable it, modify your initializer at `config/initializers/imgproxy.rb`:

```ruby
# config/initializers/imgproxy.rb

Imgproxy.extend_shrine!
```

Now you can use `imgproxy_url` method of `Shrine::UploadedFile`:

```ruby
user.avatar.imgproxy_url(width: 250, height: 250)
```

will give you an URL to your user's avatar, resized to 250x250px on the fly.

**Note:** If you use `Shrine::Storage::FileSystem` as storage, uploaded file URLs won't include the hostname, so imgproxy server won't be able to access them. To fix this, use `host` option of `Imgproxy.extend_shrine!`:

```ruby
# config/initializers/imgproxy.rb

Imgproxy.extend_shrine!(host: "http://your-host.test")
```

Alternatively, you can launch your imgproxy server with the `IMGPROXY_BASE_URL` setting:

```
IMGPROXY_BASE_URL="http://your-host.test" imgproxy
```

If you have configured both your imgproxy server and Shrine to work with Amazon S3, you may want to use the short `s3://...` source URL scheme instead of the long one generated by Rails:

```ruby
Imgproxy.extend_shrine!(use_s3: true)
```

### Usage in a framework-agnostic way

If you use another gem for your attachment operations, you like to keep things minimal or Rails-free, or if you want to generate imgproxy URLs for pictures that did not originate from your application, you can use the `Imgproxy.url_for` method:

```ruby
Imgproxy.url_for(
  "http://images.example.com/images/image.jpg",
  width: 500,
  height: 400,
  resizing_type: :fill,
  sharpen: 0.5
)
# => http://imgproxy.example.com/2tjGMpWqjO/rs:fill:500:400/sh:0.5/plain/http://images.example.com/images/image.jpg
```

You can reuse processing options by using `Imgproxy::Builder`:

```ruby
builder = Imgproxy::Builder.new(
  width: 500,
  height: 400,
  resizing_type: :fill,
  sharpen: 0.5
)

builder.url_for("http://images.example.com/images/image1.jpg")
builder.url_for("http://images.example.com/images/image2.jpg")
```

### Available imgproxy options

* `resizing_type` — defines how imgproxy will resize the image. See [URL format guide](https://github.com/imgproxy/imgproxy/blob/master/docs/generating_the_url_advanced.md#resizing-type) for available values.
* `width` — defines the width of the resulting image.
* `height` — defines the height of the resulting image.
* `dpr` — when set, imgproxy will multiply the image dimensions according to this factor for HiDPI (Retina) devices.
* `enlarge` — when true, imgproxy will enlarge the image if it is smaller than the given size.
* `extend` — when true, imgproxy will extend the image if the resizing result is smaller than the given size.
* `extend_gravity`, `extend_gravity_x`, `extend_gravity_y` — define the gravity of the extend. These options accept the same values as `gravity`, `gravity_x` and `gravity_y` (see below).
* `gravity` — defines gravity that will be used when imgproxy needs to cut some parts of the image. See [URL format guide](https://github.com/imgproxy/imgproxy/blob/master/docs/generating_the_url_advanced.md#gravity) for available values.
* `gravity_x`, `gravity_y` — specify gravity offset by X and Y axes. When `fp` gravity is used, these are floating point numbers between 0 and 1 that define the coordinates of the center of the resulting image.
* `crop_width`, `crop_height` — define the size of an area of the image to be processed (crop before resize).
* `crop_gravity`, `crop_gravity_x`, `crop_gravity_y` - define the gravity of the crop. These options accept the same values as `gravity`, `gravity_x` and `gravity_y` (see above).
* `quality` — defines the quality of the resulting image, percentage.
* `max_bytes` — when set, imgproxy automatically degrades the quality of the image until the image is under the specified amount of bytes.
* `background` — when set, imgproxy will fill the resulting image background with the specified color. Can be a hex-color string or an array of red, green and blue values (0-255).
* `brightness` — when set, imgproxy will adjust brightness of the resulting image. _Supported only by imgproxy pro._
* `contrast` — when set, imgproxy will adjust contrast of the resulting image. _Supported only by imgproxy pro._
* `saturation` — when set, imgproxy will adjust saturation of the resulting image. _Supported only by imgproxy pro._
* `blur` — when set, imgproxy will apply the gaussian blur filter to the resulting image. Value is the size of a mask imgproxy will use.
* `sharpen` — when set, imgproxy will apply the sharpen filter to the resulting image. Value is the size of a mask imgproxy will use.
* `pixelate` — when set, imgproxy will apply the pixelate filter to the resulting image. Value is the size of a pixel. _Supported only by imgproxy pro._
* `watermark_opacity` — when set, imgproxy will put a watermark on the resulting image. See [watermars guide](https://github.comimgproxym/imgproxy/blob/master/docs/watermark.md) for more info.
* `watermark_position`, `watermark_x_offset`, `watermark_y_offset`, `watermark_scale` — additional watermark options described in the [watermars guide](https://github.com/imgproxy/imgproxy/blob/master/docs/watermark.md).
* `watermark_url` — when set, imgproxy will use the image from the specified URL as a watermark. _Supported only by imgproxy pro._
* `style` — when set, imgproxy will prepend `<style>` node with provided content to the `<svg>` node of source SVG image. _Supported only by imgproxy pro._
* `preset` — array of names of presets that will be used by imgproxy. See [presets guide](https://github.com/imgproxy/imgproxy/blob/master/docs/presets.md) for more info.
* `cachebuster` — defines cache buster that doesn't affect image processing but it's changing allows to bypass CDN, proxy server and browser cache.
* `format` — specifies the resulting image format (`jpg`, `png`, `webp`).
* `base64_encode_url` — encode the URL as a base64 string.
* `use_short_options` — per-call redefinition of `use_short_options` config.

**See [imgproxy URL format guide](https://github.com/imgproxy/imgproxy/blob/master/docs/generating_the_url_advanced.md) for more info.**

### URL adapters

By default, `Imgproxy.url_for` accepts only `String` and `URI` as the source URL, but you can extend that behavior by using URL adapters.

URL adapter is a simple class that implements `applicable?` and `url` methods. See the example below:

```ruby
class MyItemAdapter
  # `applicable?` checks if the adapter can extract
  # source URL from the provided object
  def applicable?(item)
    item.is_a? MyItem
  end

  # `url` extracts source URL from the provided object
  def url(item)
    item.image_url
  end
end

# ...

Imgproxy.configure do |config|
  config.url_adapters.add MyItemAdapter.new
end
```

**Note:** `Imgproxy` will use the first applicable URL adapter. If you need to add your adapter to the beginning of the list, use the `prepend` method instead of `add`.

**Note:** imgproxy.rb provides built-inadapters for Active Storage and Shrine that are automatically added by `Imgproxy.extend_active_storage!` and `Imgproxy.extend_shrine!`.

## Further configuration

### Using truncated signature

By default, the imgproxy server uses a full-length signature (32 bytes), but you can set signature length with `IMGPROXY_SIGNATURE_SIZE` environment variable. If you have configured your imgproxy server to use truncated signatures, you need to configure the gem too:

```ruby
# config/initializers/imgproxy.rb

Imgproxy.configure do |config|
  config.signature_size = 5
end
```

### Using full processing options names

By default, imgproxy gem uses short processing options names (`rs` for `resize`, `g` for `gravity`, etc), but it can be configured to use full names:

```ruby
# config/initializers/imgproxy.rb

Imgproxy.configure do |config|
  config.use_short_options = false
end
```

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/imgproxy/imgproxy.rb.

If you are having any problems with image processing of imgproxy itself, be sure to visit https://github.com/imgproxy/imgproxy first and check out the docs at https://github.com/imgproxy/imgproxy/blob/master/docs/.

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).

## Security Contact

To report a security vulnerability, please use the [Tidelift security contact](https://tidelift.com/security). Tidelift will coordinate the fix and disclosure.
