## BigCommerce + Sincerely

### The problem

As a Bigcommerce store owner, a great way to show your appreciation is to send customers postcards with a thank you message or holiday greetings. 

### The solution

In this example, we will use Sincerely, a great postcard service. We’ll use Sincerely's web API to create a simple Ruby application, that pulls the customer data from a Bigcommerce store and sends postcards to them. In this app, we won’t focus on building a frontend — we’re just collecting merchant information via a simple Ruby script.

#### Requirements

We will use the Ruby bigcommerce gem to communicate with a Bigcommerce store. Additionally, we will use rest-client gem to communicate with the Sincerely API.
<pre>
  require 'bigcommerce'
  require 'json'
  require 'rest-client'
  require 'pp'
</pre>

#### Specifics

The initial step is to set the Bigcommerce client with API credentials and store url for whose customers we are going to send thank you cards
<pre>
store_url = 'https://store-bwvr466.mybigcommerce.com'
api_key = 'apikey'
api_user = 'username'
api = Bigcommerce::Api.new({
    :store_url => store_url,
    :username => api_user,
    :api_key => api_key
})
</pre>
Creating a postcard with Sincerely is a 2- or 3-step process. The first (two) steps are upload process, with a front photo and a logo image (optional). The photos have to PNG or a JPEG with at least 0.8 compression ratio and defined as a base64 encoded string. A successful upload will generate a photo_id, which we will use in the create card request.

<pre>
sender_info = {
      :name => "Saranyan Vigraham",
      :email => "sv@bigcommerce.com",
      :street1 => "2711 W Anderson Ln",
      :city => "Austin",
      :state => "TX",
      :postalcode => "78757",
      :country => "UNITED STATES"
}
sincerely_create_url = 'https://snapi.sincerely.com/shiplib/create'
sincerely_upload_url = 'https://snapi.sincerely.com/shiplib/upload'
sincerely_app_key = 'appkey'
front_photo = "iVBORw0KGgoAAAANSUhEUgAAAUAAAAFACAIAAABC8jL9AAAAG....="
</pre>
As you can see, we create a hash with sender info, which we ideally obtain from the merchant. If you are creating a service to send thank you notes and have a frontend/app/portal where you onboard the Bigcommerce merchants, you would ideally collect this information during registration. We are going to work with Sincerly’s create and upload endpoints. When you create a developer account and register an application, you will be provided with an app key, which we you can use here. As you can see, the photo is any image you want to use, which is base64 encoded. The next step is to generate a token that is issued with a successful image upload. In this example, we are not going to bother with the logo or profile picture and just contend with the mandatory front image.

<pre>
#upload the image to sincerely
photo_id = JSON.parse(RestClient.post sincerely_upload_url, {:photo => front_photo, :appkey => sincerely_app_key,:multipart => true})["id"]
</pre>
Now, from the store, let us retrieve the customer addresses. If a customer has multiple addresses, let us pick the first one for sake of simplicity.

<pre>
customers = api.get_customers
addresses = []
customers.each do |c|
    address = api.get_customer_addresses(c["id"])
    #check if addresses exist for the customer. if they do, pick the first one for the sake of this example to send the thank you card to.
    if address.is_a? Array
        a = address.first
        addresses << {:name => "#{a["first_name"]} #{a["last_name"]}",
                       :street1 => a["street_1"],
                       :city => a["city"],
                       :state => a["state"],
                       :postalcode => a["zip"],
                       :country => a["country"]
                    }
    end
end
</pre>
We have that out of the way. Let us create a card via sincerely for the addresses that we want to send the cards to.

<pre>
#we are going to use the same photoid for front and profile/brand images for brevity
#create a request envelope to create a card
#define test_mode => true for a test app. this will not create a card in real. refer to the sincerely api
card_request = {
  :appkey => sincerely_app_key,
  :testMode => true,
  :frontPhotoId => photo_id,
  :recipients => addresses.to_json,
  :sender => sender_info.to_json
}
resp = RestClient.post(sincerely_create_url,card_request,:accept=>:json)
pp JSON.parse(resp)
</pre>

As you can see, using a service like Sincerely to send customers thank you notes, cards, etc is very simple. What are you waiting for? Start sending cards to your customers now and show them that you care!