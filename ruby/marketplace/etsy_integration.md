Betsy		
=====
In this example, we're going to build a lightweight application that does one thing: takes products from any Bigcommerce store and publishes those products to Etsy.

For the purposes of this example app, this tool was built using Sinatra in Ruby. Feel free to clone this repo in [Github](https://github.com/iMerica/betsy) and follow along with the steps. You can also see a demo of this app at [bc-shopping-feed.herokuapp.com](http://bc-shopping-feed.herokuapp.com/) .

We also suggest creating a developer account at [developer.etsy.com](http://developer.etsy.com). Once completed, you should be given a API Key and API Secret which you'll need if you want to actually run the application locally. Also be advised that you will need a real Etsy store to use this application as a merchant would use it. 

Disclaimer
----------
Not everything is covered in this document and we suggest having a look at the code in Github to see how all moving parts come together.

Steps
-----


The first thing we need to do is create a form that allows us to capture merchants Bigcommerce API details and store them in a database.

In our index.rb file, our Sinatra code to handle this form page looks like this:



	# /index.rb
	
	require 'sinatra'
	
	#Enable sessions so we can identify the current user
	enable :sessions 
	
	get '/' do
	  erb :index  
	end


The code above tells Sinatra to listen for all incoming requests at "/" and  respond with HTML from the **index.erb**  file: 


	<!-- views/index.erb -->
	
	<div id="wrapper">
		<div class="page-header">
			<h1 style="padding-left:15px;">
				Betsy
				<img src="/images/beta-stamp.png" id="beta">
				<p class="lead"> Publish your Bigcommerce products to Etsy</p>
			</h1>
		</div>
		<div class="container">
			<center>
				<form action="auth" method="post" id="feedr">
					<label>
						Name
					</label>
						<input type="text" name="name"></input> 
					</p>
					<label>
						Email Address 
					</label>	
	          		<input type="text" name="email"></input>
					</p>
					<label>
						Bigcommerce Store URL
					</label>
						<input type="text" name="bc_store_url"></input>
					</p>
					<label>
						Bigcommerce Username
					</label>	
						<input type="text" name="bc_username"></input>
					</p>
					<label>
						Bigcommerce API Key
					</label>
						<input type="text" name="bc_api_key"></input>
					</p>	 
					<input type="submit" class="btn btn-primary" value="Authorize"></input>
				</form>
			</center>
		</div>
	</div>


  This form will post to the URI /auth. In our Sinatra route for /auth, we need to first store the user submitted details in a database, then redirect the user to Etsy so that Etsy can grant this app permission to make changes to the merchant's Etsy store. 
  
  For storage we're going to use [Sequel]("https://www.google.com.au/search?q=Sequel&oq=Sequel&aqs=chrome.0.57j60l3j59l2.1332&sourceid=chrome&ie=UTF-8") which provides an easy to use DSL for SQL queries. In this example, we're connecting to a Postgres database, but MySQL is fine too. 

Lets include that library in our application file:



	# /index.rb
		
	require 'sinatra'
	require 'sequel'

	
	To keep things organized, we'll put this db creation step in a separate file which we can call at any time by running ` $ rake create_table ` from the command line.

		
	# /Rakefile
	require 'sequel'
	
	DB = Sequel.connect(ENV['DATABASE_URL'].to_s)
	
	desc "Create the tables"
	task :create_table do
	  if DB.table_exists?(:user_info) == false
	    DB.create_table :user_info do
	      primary_key :id
	      String :name, :null => false
	      TrueClass :active, :default => true
	      DateTime :created_at
	      column :email, :text, :null => false
	      column :bc_product_data, :text
	      column :bc_username, :text
	      column :bc_store_url, :text
	      column :bc_api_key, :text
	      column :etsy_oauth_token, :text
	      column :etsy_store_id, :text
	      column :etsy_oauth_verifier, :text
	    end
	    puts "New table created successfully"
	  else
	    puts "Table already exists"    
	  end
	end



Now that we have the database setup, lets add all the Etsy client options for this application as well as some of the other dependcies.

	
	# /index.rb
	
	require 'etsy'
	require 'oauth2'
	require 'logger'
	require 'sequel'
	require 'bigcommerce'
	require 'rest_client'
	require 'rack-flash'
	require 'sinatra/redirect_with_flash' 
	
	enable :sessions
	use Rack::Flash,:sweep => true 
	
	unless KEYSTRING = ENV['KEYSTRING']
	  raise "You must specify the KEYSTRING env variable"
	end
	unless SHARED_SECRET = ENV['SHARED_SECRET']
	  raise "You must specify the SHARED_SECRET env veriable"
	end
	
	# Etsy API options
	Etsy.environment = :production if ENV['PROD_MODE'] == 'true'
	Etsy.api_key = KEYSTRING
	Etsy.api_secret = SHARED_SECRET
	Etsy.callback_url =  ENV['CALLBACK_URL']  || "localhost:3000/authorize" 
	
	DB = Sequel.connect(ENV['DATABASE_URL'] || "postgres://localhost/shopper")


	
Okay, we now have everything we need to properly handle the incoming **POST** request to */auth*.


	# /index.rb
	
	post '/auth' do	  
	  @users = DB[:user_info]
	  @users.insert(
	    :name => params[:name],
	    :email => params[:email],
	    :bc_store_url => params[:bc_store_url],
	    :bc_username => params[:bc_username], 
	    :bc_api_key => params[:bc_api_key]
	    )
	  session[:current_user] = @users.where(:email => params[:email]).all[0][:id]
	
	  request_token = Etsy.request_token
	  session[:request_token]  = request_token.token
	  session[:request_secret] = request_token.secret
	  redirect Etsy.verification_url
	end

Notice we're redirecting users to a URL called **Etsy.verification_url**. This means the merchant first goes to Etsy to grant the application access, then gets redirected back to the application (see [OAuth2](http://en.wikipedia.org/wiki/OAuth)). The URL Etsy redirects to is set by assigning a URL to **Etsy.callback_url**, but if its *example.com/authorize* then the route in sinatra would look like this:


	# /index.rb
	
	get '/authorize' do
	  @users = DB[:user_info]  
	  @access_token = params[:oauth_token]
	  @users.where(:id => session[:current_user]).update(:etsy_oauth_token => @access_token)
	  erb :success
	end

We now have everything we need to move data between Bigcommerce and Etsy. Moving long to the fun part...


	# /index.rb
	
	post '/temp' do		
	  @user_table = DB[:user_info]
	  @found_user = @user_table.where(:id => session[:current_user]).all[0]
	  
	  @bc_client = Bigcommerce::Api.new({
	    :store_url => @found_user[:bc_store_url],
	    :username => @found_user[:bc_username],
	    :api_key => @found_user[:bc_api_key]
	    })
	
	  @all_products = @bc_client.get_products()
	
	  #An array of fields we need to strip out from the bigcommerce product catalog as they're not needed 
	  @black_list = ["id", "keyword_filter", "type", "sku", "availability_description", "cost_price", 
	  "retail_price", "sale_price", "sort_order", "is_visible","is_featured", "related_products", 
	  "inventory_level","inventory_warning_level", "warranty", "weight", "width", "height", 
	  "fixed_cost_shipping_price", "is_free_shipping", "inventory_tracking", "rating_total", "rating_count",
	  "total_sold", "date_created","brand_id", "view_count", "page_title","meta_keywords", "meta_description",
	  "layout_file", "is_price_hidden","price_hidden_label", "categories", "date_modified",
	  "event_date_field_name", "event_date_type", "event_date_start", "event_date_end", "myob_asset_account",
	  "myob_income_account", "myob_expense_account", "peachtree_gl_account", "condition", 
	  "is_condition_shown", "preorder_release_date", "is_preorder_only", "preorder_message",
	  "order_quantity_minimum", "order_quantity_maximum", "open_graph_type", "open_graph_title",
	  "open_graph_description", "is_open_graph_thumbnail", "upc", "date_last_imported", "option_set_id",
	  "tax_class_id", "option_set_display", "bin_picking_number","custom_url", "availability", "brand",
	  "discount_rules", "configurable_fields", "custom_fields", "videos","skus","rules", 
	  "option_set", "options", "tax_class"]
	
	  #Strip out all the undesired fields
	  @all_products.each { | each_product | each_product.reject! { |field| @black_list.include?(field) } }
	  #Jsonize the product data
	  @jsonized_prods = @all_products.to_json
	  #Insert the JSONized data into the DB
	  @user_table.where(:id => session[:current_user]).update(:bc_product_data => @jsonized_prods)
	
	  erb :preview		  
	end
		
		
That giant array of product attributes are the fields which are used by Bigcommerce, but not by Etsy - so this whole section of code above is about filtering the data and allowing the merchant to preview it before submitting. Once they've previewed it we can take give the merchant an estimate of what Etsy is going to charge them to list there products (approximately $0.20 per product). Below is the final step


	# /index.rb
	
	post '/send' do
	  @user_table = DB[:user_info] 
	  @found_user = @user_table.where(:id => session[:current_user]).all[0]
	  
	  # Code to POST product data to Etsy's production API goes here.
	end
	
You can run the server locally by first creating the table with `rake create_table` then you can run it with: `$ KEYSTRING=xxxxxxxx SHARED_SECRET=xxxxxxxx ruby index.rb` or `$ KEYSTRING=xxxxxxxx SHARED_SECRET=xxxxxxxx thin start` if you have thin installed (replace the placeholders with your actual keys).