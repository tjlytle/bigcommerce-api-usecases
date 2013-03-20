## BigCommerce Simple Order Analytics

### Problem
This analytics application retrieves orders from a Bigcommerce store and computes basic analytics like prices and demographics.  You enter your API credentials in a webpage, the app connects to the Bigcommerce store and retrieves data to run analytics upon. 

### Requirements
This app uses the Bigcommerce gem and uses sinatra to create and stage the Ruby app.

require 'rubygems'
require 'sinatra'
require 'bigcommerce'
require 'json'
require 'haml'
require 'pp'
The main dependencies are listed above. The core ones are sinatra, json and bigcommerce. Haml gem is used for creating elegant views and pp is for pretty printing the output for console logs. We also use highcharts to display the graphs.

### Specifics

First, create an instance of the Bigcommerce client that is going to communicate with your store with the specified username and key. 
<pre>
    store_url = 'https://store-bwvr466.mybigcommerce.com'
    api_key = '567df000b3e5c0a203f42666531e16ed'
    api_user = 'testuser'
    api = Bigcommerce::Api.new({
        :store_url => store_url,
        :username => api_user,
        :api_key => api_key
    })
</pre>

Take a look at the orders endpoint for information on how to get orders from a store. When using the ruby gem, we can simply do a api.get_orders call. You can look at the sample quickstart code for fetching orders here.

<pre>
    orders = api.get_orders
    @dates = orders.collect{|o| o["date_created"]}
    @amount = orders.collect{|o| Float(o["subtotal_inc_tax"])}
    demographics = orders.collect{|o| o["billing_address"]["state"]}
</pre>

The <code>get_orders</code> call returns a list of all orders, when used without parameters. You can for instance, get a list of orders matching a criteria by simply adding a parameters hash to the request. 
<pre>
api.get_orders({:status_id => 10})
</pre>

Once we have the set of orders we need, we will collect the dates on which the orders were placed and their sub totals. This will let us plot a simple chart to aggregate the revenue. We also intend to show a breakdown of customer demographics, helpful for a merchant to determine in which locations the business is doing well. For simplicity and brevity, we are going to assume that billing address of the customer who placed the order gives us an accurate representation of this data.

We can represent the demographics using a pie chart. Let us use Highcharts to display this data. If we take a look at the demo apps for Highcharts, we will see that the pie charts data is represented as array of arrays-  [[a,3],[b,2],[c,5]]

<pre>
@pie = [] 
a = Hash.new{|h,k| h[k] = 0}
demographics.each do |d|
   a[d] = a[d] + 1
end

a.keys.each do |k|
   @pie << [k,a[k]]
end

</pre>

Using code as above, we can create the array of values we need. Now, we have the data we need to display the charts. In our haml view, we need to render the charts using Javascript.

<pre>
:javascript

chart = new Highcharts.Chart({
chart: {
    renderTo: 'chart',
    type: 'line',
    marginRight: 130,
    marginBottom: 25
    },
title: {
    text: 'Orders'
    },
xAxis: {
    categories: #{@dates}
    },
yAxis: {
    title: {
        text: 'Sale price'
        },
    },
series: [{
    name: 'Signature products',
    data: #{@amount}
    }]
    });

dchart = new Highcharts.Chart({
chart: {
    renderTo: 'demog'
    },
title: {
    text: 'Customer demographics'
    },
series: [{
    type: 'pie',
    name: 'Where do your customers come from',
    data: #{@pie}

    }]
})
</pre>
This code will render the charts to div elements #chart and #demog. This is our simple analytics app that shows order information graphically. 