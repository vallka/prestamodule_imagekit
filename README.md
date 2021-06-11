![](https://www.vallka.com/media/markdownx/2021/06/09/e25ed155-fc6e-4d44-a691-7abeace6767d.png)

Recently we found a really useful service - [imagekit.io](https://imagekit.io/).

> *Image CDN with automatic optimization, real-time transformation, and storage that you can integrate with existing setup in minutes.*

What is the most amazing thing about this service, it has a free plan with very a generous set of features and allowances.

Let's use it for a Prestashop theme.

First of all, we need to create an account on [imagekit.io](https://imagekit.io/). After verifying your email, you'll be able to login to your account. At the very first step you will be asked for IMAGEKIT ID. By default this is a random string value. It make sense to change this random string to something meaningful, like the main portion of your website domain name. After setting all up, you will be unable to change your IMAGEKIT ID, so please do it right now.

Next, the imagekit terminology becomes a somewhat confusing. We need to **"Attach existing origin in ImageKit.io"**. Click on the Attach existing origin in ImageKit.io link and then click on **ADD NEW ORIGIN** button. **Origin Name** may be any string, for **Origin Type** select *Web Folder - HTTP(S) server and Magento, Shopify, Wordpress, etc.*. For **Base URL** use ***base*** URL, like https://www.example.com (without /images subfolder as it advise you). Leave the bottom checkboxes unchecked (unless you know what they are), and click Submit.

What has confused me a little - where can you find this newly created "origin" after it has been set up? Under "URL endpoins" menu at the left. You will find  "Default URL endpoint" there. Click on it, here you can edit your "origins". You might need a new "origin", if you are using a multi-store feature of Prestashop. For example, we have 2 shops and 2 domains for these shops in our Prestashop - gellifique.co.uk for UK shop and gellifique.eu for Spanish shop. Both domains should be added to Imagekit Default URL endpoins as 2 separate "origins". One endpoint, two origins. This was somewhat confusing.

Basically, all done. Now we can use ImageKit URLs instead of original URLs for images in your theme. I wanted to pinpoint all templates in Classic theme which needs to be updated, but as it happened, the Classic theme has changed a lot since we used it to create a custom theme for our webshop. In our case, I had to modify these files (under /themes/mytheme/ folder):


```
templates/catalog/listing/category.tpl 
templates/catalog/_partials/product-cover-thumbnails.tpl 
templates/catalog/_partials/product-images-modal.tpl 
templates/catalog/_partials/miniatures/product.tpl 
modules/ps_imageslider/views/templates/hook/slider.tpl 
```

In the latest version of the Classic theme some of these files became split into two or more files and some references may have moved somewhere else... So I'll show an example of what I did to my, probably  outdated version. Let's take this file:

```
templates/catalog/_partials/miniatures/product.tpl 
```

It looks like it didn't change much. Let's find a reference to a product image (lines 33-37):

```
<img
  src="{$product.cover.bySize.home_default.url}"
  alt="{if !empty($product.cover.legend)}{$product.cover.legend}{else}{$product.name|truncate:30:'...'}{/if}"
  data-full-size-image-url="{$product.cover.large.url}"
/>
```

We need to change src attribute - with what? PHP parse_url() function will help us:

```
src = "https://ik.imagekit.io/ourimagekitid{parse_url($product.cover.bySize.home_default.url, PHP_URL_PATH)}"
```

where ourimagekitid is the IMAGEKIT ID we have set up earlier. If we know the exact size of the image here, we may specify it for the Imagekit, for example:
```
src = "https://ik.imagekit.io/ourimagekitid/tr:w-250,h-250{parse_url($product.cover.bySize.home_default.url, PHP_URL_PATH)}"
```

Do similar exercises to remaining templates, and the task is completed! However...

We had to hardcode IMAGEKIT ID into several files. Not good. Next issue - sometimes you may want to switch off ImageKit completely, for example in development environment. And the last one (at least for me) - although ImageKit gives us an ability to Purge cache, we have to specify it file by file, and even so it doesn't occure immediately. Sometimes you have to wait a few minutes before it have an effect... Would be nice to implement some feature like file versioning, like attaching ?<random string> to the end of the URL.
	
So let's write another Prestashop module.
	
Like before, we will create a module skeleton, using ([see my first article](https://www.vallka.com/blog/prestashop-modules-programming-bcc-outgoing-emails/)):

https://validator.prestashop.com/generator
	
We will register two hooks:
	
```
public function install()
{
    Configuration::updateValue('IMAGEKIT_LIVE_MODE', false);
    return parent::install() &&
            $this->registerHook('actionDispatcher') && 
            $this->registerHook('backOfficeHeader');
}
	
```
	
"backOfficeHeader" hook is needed only for attaching some js file to back office (not really important):
	
```
public function hookBackOfficeHeader()
{
    if (Tools::getValue('configure') == $this->name || Tools::getValue('module_name') == $this->name) {
        $this->context->controller->addJS($this->_path.'views/js/back.js');
    }
}
```

And the important one is hookActionDispatcher. It allows us to define a custom plugin for Smarty templates. This plugin can be used as a function in a Smarty template
	
```
public function hookActionDispatcher(){
    $this->context->smarty->registerPlugin('function', 'imagekit', array('Imagekit', 'imagekitUrl'));
}
```

And the function implementation construct the needed URL for ImageKit image:	
	
```
public static function imagekitUrl($params,$smarty) {
    if ($params['url']) {
        if ( Configuration::get('IMAGEKIT_LIVE_MODE', false) && Configuration::get('IMAGEKIT_ENDPOINT', '')) {
            $tr = $params['tr'] ? 'tr:'.$params['tr'] : '';
            $v = $params['v'] ? '?'.Configuration::get('IMAGEKIT_TS', '') : '';
              return Configuration::get('IMAGEKIT_ENDPOINT', '') . $tr . parse_url($params['url'],PHP_URL_PATH) . $v;
        }

        return $params['url'];
    }
    return '';
}
```
	
It takes configuration parameters from the module configuration. IMAGEKIT_LIVE_MODE allows us to switch using ImageKit on/off with a single click. Parameters passed to the function are "url", "tr", and "v" (only "url" parameter is mandatory). "tr" set required transformation in ImageKit format, and v=1 (0 by default) append timestamp "?<timestamp>" querystring to the URL.
	
So now the src attribute in templates/catalog/_partials/miniatures/product.tpl template will look like this:
	
```
src = "{imagekit tr='w-250,h-250' url=$product.cover.bySize.home_default.url}"
```
	
What about "v" (versioning) parameter and why all the buzz? As I found out, the only place where it makes sense is templates/catalog/listing/category.tpl  template. This is the only place where a newer image is being saved by the same name as a previous one. If you upload a new image for a category with id 25, for example, the image URL will be /img/c/25.jpg (if you upload a jpg file). If you upload a new image, it still will have this name. Thus ImageKit will think it is the same image. You can Purge cache from ImageKit dashboard (providing you know the exact URL), wait some time and the change will happen... But for me it was easier to introduce "versioning". You have to go to my module configuration, double-click on Version (Timestamp) field, ans save configuration. And image URL:
	
```
/img/c/25.jpg
```

will become something like this
	
```
/img/c/25.jpg?1623440107030

```
	
and ImageKit will treat it as a new image.
	
In all other templates Prestashop generates a unique image name, using either the actual file name, or some random values, and such problem doesn't raise.
	
So in the file 	templates/catalog/listing/category.tpl
	
the line
	
```
<img class="replace-2x" 
    src="{$link->getCatImageLink($subcategory.link_rewrite,$subcategory.id_image,'')}"
    alt="{$subcategory.name|escape:'html':'UTF-8'}"/>
```

became	

```
<img class="replace-2x" 
    src="{imagekit tr='w-230,h-230' v=1 url=$link->getCatImageLink($subcategory.link_rewrite,$subcategory.id_image,'')}"
    alt="{$subcategory.name|escape:'html':'UTF-8'}"/>
```
	
I cannot find a similar line in the modern version of Classic theme. Maybe subcategories images are not shown at all in the modern Classic theme?
	
Ok, everything is working now as I wanted it. Except for a small problem what I couldn't solve. Maybe some of the readers will point me to a right direction ?
	
If, for some reason, I will unregister 	 or disable my module, all images will go! Because Smarty plugin "imagekit" doens't exist any more. I wanted to do something like this in my templates:
	
```
<img class="replace-2x" 
		{if function_exists("imagekitUrl")}
		    src="{imagekit tr='w-230,h-230' v=1 url=$link->getCatImageLink($subcategory.link_rewrite,$subcategory.id_image,'')}"
		{else}
		    src="{$link->getCatImageLink($subcategory.link_rewrite,$subcategory.id_image,'')}"
	 {/if}
		 />
```

but for some reason it doesn't seem to work. Anyone can help? (Maybe I had to use "modifer" instead of "function" in Smarty plugin, it would be easier than?)
	
