# CropTool

application.html.slim

```ruby
= yield :crop_tool
```

CSS

```css
#= require crop_tool/crop_tool
#= require crop_tool/jcrop/jquery.Jcrop
```

javascript

```coffee
#= require crop_tool/crop_tool
#= require crop_tool/jcrop/jquery.Jcrop

$ ->
  CropTool.init()
```

views

```slim
  ruby:
    crop_data_base = {
      url:   url_for([:main_image_crop_base, object]),
      source: object.main_image.url(:original),
      holder:  { width: 500 },
      preview: { width: 270, height: 210 },
      final_size: "270x210",
      callback_handler: "CropTool.pub_main_image_crop"
    }

    crop_data_preview = {
      url:   url_for([:main_image_crop_preview, object]),
      source: object.main_image.url(:original),
      holder:  { width: 500 },
      preview: { width: 100, height: 100 },
      final_size: "100x100",
      callback_handler: "CropTool.pub_main_image_crop"
    }

  = link_to "Обрезать 270x210", "#", class: "btn btn-size-m mr15 js_crop_tool", data: crop_data_base
  = link_to "Обрезать 100x100", "#", class: "btn btn-size-m mr15 js_crop_tool", data: crop_data_preview
  = link_to 'Поворот', [:main_image_rotate, object], method: :patch,  class: 'btn btn-size-m'
```

controller

```ruby
module MainImageForObjectActions
  ACTIONS = %w[
    main_image_crop_base
    main_image_crop_preview
    main_image_rotate
    main_image_delete
  ]

  def main_image_crop_base
    path = @main_image_holder.main_image_crop_base(params)
    render json: { ids: { main_image_base_pic: path } }
  end

  def main_image_crop_preview
    path = @main_image_holder.main_image_crop_preview(params)
    render json: { ids: { main_image_preview_pic: path } }
  end

  def main_image_rotate
    @main_image_holder.main_image_rotate
    redirect_to :back
  end

  def main_image_delete
    @main_image_holder.main_image_destroy!
    redirect_to :back
  end
end

```

controller

```ruby
class ProductsController < ApplicationController
  include MainImageForObjectActions
  before_action :authenticate_user!

  before_action :set_product, only: [:show, :edit, :update, :destroy] + MainImageForObjectActions::ACTIONS
  before_action :set_holder_for_main_image, only: MainImageForObjectActions::ACTIONS
end
```

routes.rb

```ruby
  resources :products do
    member do
      patch  :main_image_crop_base
      patch  :main_image_crop_preview
      patch  :main_image_rotate
      delete :main_image_delete
    end
  end
```

Model

```ruby
  def main_image_crop_base params
    crop_params = params[:crop].symbolize_keys

    src  = main_image.path
    dest = main_image.path :base

    manipulate({ src: src, dest: dest }.merge(crop_params)) do |image, opts|
      scale = image[:width].to_f / opts[:img_w].to_f
      image = crop image, opts[:x], opts[:y], opts[:w], opts[:h], scale
      image = strict_resize image, 270, 210
      image
    end

    main_image.url(:base, timestamp: false)
  end

  def main_image_crop_preview params
    crop_params = params[:crop].symbolize_keys

    src  = main_image.path
    dest = main_image.path :preview

    manipulate({ src: src, dest: dest }.merge(crop_params)) do |image, opts|
      scale = image[:width].to_f / opts[:img_w].to_f
      image = crop image, opts[:x], opts[:y], opts[:w], opts[:h], scale
      image = strict_resize image, 100, 100
      image
    end

    main_image.url(:preview, timestamp: false)
  end

  def main_image_rotate
    return false unless main_image?

    src     = main_image.path
    base    = main_image.path :base
    preview = main_image.path :preview

    [src, base, preview].each do |image_path|
      manipulate({ src: image_path, dest: image_path }) do |image, opts|
        rotate_right image
      end
    end
  end

  def main_image_destroy!
    base    = main_image.path(:base).to_s
    preview = main_image.path(:preview).to_s

    destroy_file [base, preview]
    main_image.destroy

    save!
  end
```
