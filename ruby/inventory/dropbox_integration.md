If you don’t know dropbox, check it out. It gives you a way to store your files on the cloud and sync it up with different devices. One problem a lot of merchants/ecommerce clients run into is irregular backups. Sometimes, you get hosed when there is data corruption and stuff is not backed up. There are several ways in which you can backup Bigcommerce store data via WebDAV. A different route is presented here that is more flexible and can be tailored to backup just specific data to dropbox. The example that is shown here is presented as a Ruby rake task that you can use/modify to schedule your own backups.

```
require 'bigcommerce'
require 'httparty'
require 'dropbox-api'
 
namespace :backups  do
  desc "Run a backup of all images"
  task :run => :environment do
 
    # let us setup a user with bcstore and dropbox creds
    # this assumes that the user has allowed access to the dropbox 
    # application registered and in return 
    # we are storing their accesskey and secret.
    # to follow the oauth flow and grab the tokens, please refer to the doc at 
    # https://github.com/futuresimple/dropbox-api
 
    user = {
      "bcstore" => {
        "store_url" => "store-xxx.mybigcommerce.com",
        "user_name" => "admin",
        "api_key" => "jfkdj34..."
      },
      "dbstorage" => {
        "access_token" => "db access token here", 
        "secret" => "db secret key"
      }
    }
 
    # configure your app - the app_key and secret is provided when you register your app with dropbox
    Dropbox::API::Config.app_key    = "appkey"
    Dropbox::API::Config.app_secret = "appsecret"
    Dropbox::API::Config.mode       = "sandbox"
 
    # just a pointless check, but illustrated as a case, if the user's data is being fetched from database
    # to proceed, we need valid bigcommerce store creds and dropbox oauth access tokens
 
    if !user.bcstore.nil? && !user.dbstorage.nil?
      
      client = Dropbox::API::Client.new(:token => user["dbstorage"]["access_token"], :secret => user["dbstorage"]["secret"])
      api = Bigcommerce::Api.new({
        :store_url => user["bcstore"]["store_url"],
        :username  => user["bcstore"]["user_name"],
        :api_key   => user["bcstore"]["api_key"]
      })
 
      # let us get 200 images, the max allowed and update the counter
      images = api.get_products_images({:limit => 200, :page => 1})
      count = 0
      page = 1
 
      loop do
        image = images[count]
        img_url = user.bcstore.store_url + "/product_images/" + image["image_file"]
 
        # download the image to dropbox
        response = HTTParty.get(img_url)
        client.upload image['image_file'], response.body
        puts img_url
 
        count++
 
        # verbose check, provided for clarity
        # images < 200, means you have hit the last batch.
        break if (images.size < 200) and (count == images.size - 1)
 
        # else
        page++
        images = api.get_products_images({:limit => 200, :page => page})
        end
      end
    end
  end
end
```
To get started, you need to <a href="https://www.dropbox.com/developers/apps" title="create an app with dropbox" target="_blank">register an app with Dropbox</a>. If you have a rails application this code is going to be part of, you can go through a standard oauth flow to obtain access token and secret for your app to work with. Essentially, without the access_token and secret, the app will not be able to interact with your dropbox, even though, you might own the app. There are multiple ways you can generate that access token. We are not going to go into details on that. But, please take a look at the dropbox-api gem <a href="https://github.com/futuresimple/dropbox-api" title="dropbox-api" target="_blank">(<a href="https://github.com/futuresimple/dropbox-api">https://github.com/futuresimple/dropbox-api</a></a>), which is what we are using in this example. There are examples on getting an access token either via a Rails app or a rack based app.

We will be using the bigcommerce client library for Ruby found <a href="https://github.com/saranyan/bigcommerce-api-ruby" title="bigcommerce api ruby" target="_blank">here</a>.

1. Step 1 would be configure the dropbox client using the application credentials.

```
Dropbox::API::Config.app_key    = "appkey"
Dropbox::API::Config.app_secret = "appsecret"
Dropbox::API::Config.mode       = "sandbox" 
client = Dropbox::API::Client.new(:token => user["dbstorage"]["access_token"], :secret => user["dbstorage"]["secret"])
```
2. Configure the Bigcommerce api client
```
api = Bigcommerce::Api.new({
        :store_url => user["bcstore"]["store_url"],
        :username  => user["bcstore"]["user_name"],
        :api_key   => user["bcstore"]["api_key"]
    })
```
3. Query the product images and upload them to dropbox. The Bigcommerce store has product images under a directory called product_images. When you query a product image, the object model looks like the following -

```
{
    "id": 117,
    "product_id": 30,
    "image_file": "d/282/astonishing-x-men-1-100k__08036.jpg",
    "is_thumbnail": true,
    "sort_order": 0,
    "description": "",
    "date_created": "Fri, 21 Dec 2012 19:00:47 +0000"
}
```
The image_file gives the directory (or relative path) of the image. The full path can be obtained by store_url/product_images/image_file_path. This can be simply uploaded to dropbox using 

  client.upload_image 'filename', 'file'. 

As seen in the code above, we can fetch the image via HTTP get and upload it to dropbox.
This is a quick and cheap way to store product images that are synced up and ready to go. You can easily modify this script to schedule backups via cron jobs or something of that end or even build a backup service powered by dropbox. 

