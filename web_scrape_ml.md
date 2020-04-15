## TL;DR
Uses the crawly web scrapping api and tensorflex
### Article Link
erlang-solutions.com/blog/how-to-build-a-machine-learning-project-in-elixir.html
### Author
Grigory Starinkin and Oleg Tarasenko
## Key Takeaways
* Extracts the data from a group of websites (in this case we will demonstrate how to extract data from the harveynorman.ie shop)
* Train a neural network to recognise a product category from the product image
* Integrate the neural network into the Elixir code so it completes the image recognition and suggests products
* Build a web app which glues everything together. 

## Useful Code Snippets

lib/products_advisor/spiders/harveynorman.ex
```elixir
defmodule HarveynormanIe do
 @behaviour Crawly.Spider

 require Logger
 @impl Crawly.Spider
 def base_url(), do: "https://www.harveynorman.ie"

 @impl Crawly.Spider
 def init() do
  [
   start_urls: [
    "https://www.harveynorman.ie/tvs-headphones/"
   ]
  ]
 end

 @impl Crawly.Spider
 def parse_item(response) do

    # Extracting pagination urls
  pagination_urls =
   response.body |> Floki.find("ol.pager li a") |> Floki.attribute("href")

    # Extracting product urls
  product_urls =
   response.body |> Floki.find("a.product-img") |> Floki.attribute("href")

  all_urls = pagination_urls ++ product_urls

    # Converting URLs into Crawly requests
  requests =
   all_urls
   |> Enum.map(&build_absolute_url/1)
   |> Enum.map(&Crawly.Utils.request_from_url/1)

  # Extracting item fields
  title = response.body |> Floki.find("h1.product-title") |> Floki.text()
  id = response.body |> Floki.find(".product-id") |> Floki.text()

  category =
   response.body
   |> Floki.find(".nav-breadcrumbs :nth-child(3)")
   |> Floki.text()

  description =
   response.body |> Floki.find(".product-tab-wrapper") |> Floki.text()

  images =
   response.body
   |> Floki.find(" .pict")
   |> Floki.attribute("src")
   |> Enum.map(&build_image_url/1)

  %Crawly.ParsedItem{
   :items => [
    %{
     id: id,
     title: title,
     category: category,
     images: images,
     description: description
    }
   ],
   :requests => requests
  }
 end

 defp build_absolute_url(url), do: URI.merge(base_url(), url) |> to_string()

 defp build_image_url(url) do
  URI.merge("https://hniesfp.imgix.net", url) |> to_string()
 end

end

```

Jaypeg
```elixir
defmodule Jaypeg do
  @moduledoc 
  Simple library for JPEG processing.

  ## Decoding

   elixir
  {:ok, <<104, 146, ...>>, [width: 2000, height: 1333, channels: 3]} =
      Jaypeg.decode(File.read!("file/image.jpg"))



  @on_load :load_nifs

  @doc 
  Decode JPEG image and return information about the decode image such
  as width, height and number of channels.

  ## Examples

      iex> Jaypeg.decode(File.read!("file/image.jpg"))
      {:ok, <<104, 146, ...>>, [width: 2000, height: 1333, channels: 3]}


  def decode(_encoded_image) do
    :erlang.nif_error(:nif_not_loaded)
  end

  def load_nifs do
    :ok = :erlang.load_nif(Application.app_dir(:jaypeg, "priv/jaypeg"), 0)
  end
End
```

nif 
```elixir
static ERL_NIF_TERM decode(ErlNifEnv *env, int argc,
                           const ERL_NIF_TERM argv[]) {
  ERL_NIF_TERM jpeg_binary_term;
  jpeg_binary_term = argv[0];
  if (!enif_is_binary(env, jpeg_binary_term)) {
    return enif_make_badarg(env);
  }

  ErlNifBinary jpeg_binary;
  enif_inspect_binary(env, jpeg_binary_term, &jpeg_binary);

  struct jpeg_decompress_struct cinfo;
  struct jpeg_error_mgr jerr;
  cinfo.err = jpeg_std_error(&jerr);
  jpeg_create_decompress(&cinfo);

  FILE * img_src = fmemopen(jpeg_binary.data, jpeg_binary.size, "rb");
  if (img_src == NULL)
    return enif_make_tuple2(env, enif_make_atom(env, "error"),
                            enif_make_atom(env, "fmemopen"));

  jpeg_stdio_src(&cinfo, img_src);

  int error_check;
  error_check = jpeg_read_header(&cinfo, TRUE);
  if (error_check != 1)
    return enif_make_tuple2(env, enif_make_atom(env, "error"),
                            enif_make_atom(env, "bad_jpeg"));

  jpeg_start_decompress(&cinfo);

  int width, height, num_pixels, row_stride;
  width = cinfo.output_width;
  height = cinfo.output_height;
  num_pixels = cinfo.output_components;
  unsigned long output_size;
  output_size = width * height * num_pixels;
  row_stride = width * num_pixels;

  ErlNifBinary bmp_binary;
  enif_alloc_binary(output_size, &bmp_binary);

  while (cinfo.output_scanline < cinfo.output_height) {
    unsigned char *buf[1];
    buf[0] = bmp_binary.data + cinfo.output_scanline * row_stride;
    jpeg_read_scanlines(&cinfo, buf, 1);
  }

  jpeg_finish_decompress(&cinfo);
  jpeg_destroy_decompress(&cinfo);

  fclose(img_src);

  ERL_NIF_TERM bmp_term;
  bmp_term = enif_make_binary(env, &bmp_binary);
  ERL_NIF_TERM properties_term;
  properties_term = decode_properties(env, width, height, num_pixels);

  return enif_make_tuple3(
    env, enif_make_atom(env, "ok"), bmp_term, properties_term);
}
```

Image resizing nif
```elixir
static ERL_NIF_TERM resize(ErlNifEnv *env, int argc,
                           const ERL_NIF_TERM argv[]) {
  ErlNifBinary in_img_binary;
  enif_inspect_binary(env, argv[0], &in_img_binary);

  unsigned in_width, in_height, num_channels;
  enif_get_uint(env, argv[1], &in_width);
  enif_get_uint(env, argv[2], &in_height);
  enif_get_uint(env, argv[3], &num_channels);

  unsigned out_width, out_height;
  enif_get_uint(env, argv[4], &out_width);
  enif_get_uint(env, argv[5], &out_height);

  unsigned long output_size;
  output_size = out_width * out_height * num_channels;
  ErlNifBinary out_img_binary;
  enif_alloc_binary(output_size, &out_img_binary);

  if (stbir_resize_uint8(
        in_img_binary.data, in_width, in_height, 0,
        out_img_binary.data, out_width, out_height, 0, num_channels) != 1)
    return enif_make_tuple2(
      env,
      enif_make_atom(env, "error"),
      enif_make_atom(env, "resize"));

  ERL_NIF_TERM out_img_term;
  out_img_term = enif_make_binary(env, &out_img_binary);

  return enif_make_tuple2(env, enif_make_atom(env, "ok"), out_img_term);
}
```

Classify Image
```elixir
def classify_image(image, graph, labels) do
    {:ok, decoded, properties} = Jaypeg.decode(image)
    in_width = properties[:width]
    in_height = properties[:height]
    channels = properties[:channels]
    height = width = 224

    {:ok, resized} =
      ImgUtils.resize(decoded, in_width, in_height, channels, width, height)

    {:ok, input_tensor} =
      Tensorflex.binary_to_matrix(resized, width, height * channels)
      |> Tensorflex.divide_matrix_by_scalar(255)
      |> Tensorflex.matrix_to_float32_tensor({1, width, height, channels})

    {:ok, output_tensor} =
      Tensorflex.create_matrix(1, 2, [[length(labels), 1]])
      |> Tensorflex.float32_tensor_alloc()

    Tensorflex.run_session(
      graph,
      input_tensor,
      output_tensor,
      "Placeholder",
      "final_result"
    )
  end
  ```

## Useful Tools
* Jaypeg
* 

## Comments/ Questions
* Uses nif
