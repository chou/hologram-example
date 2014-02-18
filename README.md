#Using Hologram with Rails to Auto-Generate Styleguides
<br>
Styleguides are handy because they document conventions for projects; making communication easier across the team whether it's between PM and dev, client and consultant, or front-endand back-end. Transitions are a little easier with declared standards already in place (pair rotation, onboarding new hires, transfer of responsibilities). From a technical perspective, styleguides keep the front end DRY with project standards and decrease the frequency of visual inconsistencies.

Despite the benefits styleguides offer, the rote documentation of CSS isn't exactly an inspiring challenge. So the team at Trulia made a gem called [Hologram]("http://trulia.github.io/hologram/") that parses and documents CSS for you.  
The setup isn't always straightforward, especially with Rails; but after setting up Hologram with some help from Guard, no additional work is needed to maintain the styleguide.
<br><br>
##Overview

Hologram parses your assets (sass, less, css, md, styl, js) for comments of the following format and generates an .html file for each category of component using the _header.html and _footer.html partials Hologram provides.

    /*doc
    ---
    title: Alert
    name: alert
    category: alerts
    ---
    ```html_example
        <div class='alert'>Hello</div>
    ```
    */
    
    .alert{
      color: blue;
    }

From the comments above, Hologram will create a file called alerts.html that has one component, .alert, and inserts an html snippet demonstrating its usage.

The stock output is pretty bare-bones; so we wanted to give it some styling. This is where Rails and its asset pipeline became too helpful--it takes some manual configuration to sequester the styleguide styles so that they don't pollute the overall app styling. We chose to remove require_tree from the manifest files and include components individually.

Here are step-by-step instructions.
<br><br>
##Installation

###Gems
In Gemfile, add 
```ruby
gem 'hologram', github: 'trulia/hologram'
```
(At the time of writing, it's necessary to specify the github repo because it's ahead of the published gem and allows some functionality that we later leverage: using categories in the _header partial.)

Hologram doesn't watch your stylesheets for changes, so to avoid having to run it manually after changes, use Guard:  

In Gemfile, add 
```ruby
gem 'guard-hologram', github: "kmayer/guard-hologram", require: false
```
(Have to specify repo because the gem wasn't pushed.)  

`bundle` to install the gems.

###Hologram
`bundle exec hologram init` will generate hologram_config.yml in the project root. This file contains five settings:

+ source:  
This is where you specify the assets that will be documented by Hologram. It's recursive; and for standard Rails apps, it makes sense to just list  `./app/assets`

+ destination:  
This is the directory that Hologram outputs to. We use `./public/styleguide`.

+ documentation_assets:  
This is where Hologram reads from in order compile styleguide files. (Most important are the _header.html and _footer.html partials that are generated after `hologram init`.)  
We left this at the default `./doc_assets`.

+ dependencies:
Left this one blank and included assets in source.

+ index:  
Hologram will create an .html file for each category of components documented; setting an index just tells Hologram which category to use as the index.  
Left it at the default `basics`.

###Guard-Hologram
Shell `guard init`, which will add Hologram to your Guardfile.
Change the Hologram part of the Guardfile to be
``` ruby
./Guardfile  

guard "hologram", config_path: "hologram_config.yml" do
  watch(%r{app/assets/stylesheets/.*css})
  watch(%r{app/assets/javascripts/.*js})
  watch(%r{doc_assets/*})
  watch('hologram_config.yml')
end
```

Now Guard will recompile your styleguide whenever the components in app/assets are changed once you run Guard.  
<br>
Creating the Styleguide
--
###Using Hologram

The syntax Hologram parses in order to include components in the styleguide is

	/*doc
	---
	title: Alert
	name: alert
	category: basics
	---
	```html_example
   	<div class='alert'>Hello</div>
	```
	*/
    
	.alert{
  		color: blue;
	}  

This tells Hologram what title to list the component under ("Alert"), the name used to apply it ("alert"), and the category it falls in ("basics").
###Initial Run
`bundle exec guard` to start watching your assets for changes. Now you won't have to worry about manually running Hologram all the time.  
To see it work, document one element of your styles in the above format, then wait for Guard to run to see what Hologram puts together in `./public/styleguide`. According to the configuration from above, Hologram will construct files for each category, plus an index, using `./doc_assets/_header.html` and `./doc_assets/_.footer.html` and place it in `./public/styleguide`. 

If you copy/paste the above example, you'll get a file called basics.html and a file called index.html in ./public/styleguide; both will have the same content since in ./hologram_config.yml we set index to point to basics.  
`rails s`, and you'll see that localhost:3000/styleguide directs you to the constructed styleguide!  

That's awesome, but since the team at Trulia wanted you to be able to customize the look of your styleguide, they left it pretty bare. Let's make it look better.
<br><br>
##Customizing the Styleguide 

We wanted to mimic the styling [demo-ed here](http://trulia.github.io/hologram-example/). To get there, we need to make the following changes:  

+ Add categories to a top-level navigation bar
+ Add styling to color code snippets
+ Add some styling to the styleguide elements (but not let these confound the overall app's styles).

The first two are easy to accomplish; just grab the code from [Trulia's Hologram Example](http://trulia.github.io/hologram-example/).  
The last point we achieve by changing the application's CSS manifest to include files individually and creating a styleguide.css.scss that holds styleguide-specific styles. Your styles should be namespaced anyway, but this approach makes sure there is no overlap.
###Getting Trulia's Styles
####Stylesheets
Grab [docs.css](https://raw.github.com/trulia/hologram-example/master/templates/static/css/doc.css) and [github.css](https://raw.github.com/trulia/hologram-example/master/templates/static/css/github.css) as well as [screen.css](https://raw.github.com/trulia/hologram-example/master/build/css/screen.css) and put them all in ./app/assets/stylesheets.  
The first two are style_guide-specific styling (github.css colors code fragments, and doc.css styles Hologram-generated classes like container.  
screen.css will serve as the styles for the app itself.
####HTML
In `./doc_assets/_header.html`, replace the code with the below code snippet, which is adapted from [Trulia's Hologram Example](https://github.com/trulia/hologram-example/blob/master/templates/_header.html).  

The modifications made were:  

+ changing stylesheet and script inclusion to require files compile by the Rails asset pipeline
+ adding a loop to auto-generate rather than hard-code links to each category.
  
(Dynamically creating category links is where using the Hologram gem from the Github repo comes into play--in the version pushed to RubyGems, we don't have access to categories in partials.)  

```html
./doc_assets/_header.html

<!doctype html>
<html>
    <head>
    	<title>HologramExample</title>
    	<link rel=stylesheet href="../assets/application.css" type="text/css">
	    <link rel=javascript href="../assets/application.js" type="text/javascript">
	</head>
	<header class="header pbn" role="banner">
    	<div class="backgroundHighlight typeReversed1">
	        <div class="container">
    	        <h1 class="h2 mvs">My Styleguide</h1>
        	</div>
	    </div>
    	<div class="backgroundLowlight typeReversed1">
        	<div class="container">
	          <span>
    	        <ul class="docNav listInline">
        	        <% for c in categories %>
            	    	<li><a href="/styleguide/<%= c[0] %>.html"><%= c[0] %></a></li>
                	<% end %>
	            </ul>
    	      </span>
        	</div>
	    </div>
	</header>
	<div class="content">
    	<section>
        	<div class="line">
            	<div class="col cols4">
                	<div class="componentMenu box boxBasic backgroundBasic">
                   		<div class="boxBody pan">
                       		<ul class="componentList listBorderedHover">
                            	<% blocks.each do |block| %>
	                            <li><a href="#<%= block[:name] %>"><%= block[:title] %></a></li>
    	                        <% end %>
        	                </ul>
            	        </div>
                	</div>
	            </div>
    	        <div class="main col cols20 lastCol">
```  
<br>
You'll notice that the header leaves a few tags open. Close them in `./doc_assets/_footer.html`:  
``` html
	      </div>
        </div>
      </section>
    </div>
  </body>
</html>
```  
<br>
Fire up `rails s` and check it out at /styleguide. It should index to whatever you specified earlier in `hologram_config.yml`. This is great, but there's one concern: the styleguide-specific styling (docs.css, github.css) are compiled into application.css per default Rails asset pipeline behavior, which means that it could leak into your core application styles if you're not careful about namespacing CSS classes.

###Isolating Styleguide Styles
####Application Styles
To ensure that the styleguide styles don't pollute the rest of the app's stylesheets, rip out the default `require_tree` from `./app/assets/stylesheets/application.css` and add each .css file manually. This allows you to specify only the application's styles, omitting the styleguide styles.   
In this example, the application's styles are contained in alert.css.scss, bubble.css.scss, tooltip.css.scss, and screen.css; so we get this:
```css
./app/assets/stylesheets/application.css.scss

@import 'alert';
@import 'bubble';
@import 'tooltip';
@import 'screen';
```
Now we need to set up a styleguide.css that will be used only by the styleguide.
####Styleguide Styles
Add a styleguide.css.scss to `./app/assets/stylesheets` and include the styleguide-specific styles. In the example, it looks like this:  
```css
./app/assets/stylesheets

@import 'docs';
@import 'github';
```
####Including Styleguide Styles
Back in the styleguide header, add 
```html
./doc_assets/_header.html

<link rel=stylesheet href="../assets/styleguide.css" type="text/css">
```
to the stylesheet inclusion to grab the new styleguide.css. This applies to all styleguides since this header will be used by all files generated by Hologram.  

####Making the Styleguide Available in Production
One more step is required to make the styleguide styles available in production. We need to tell Rails to precompile styleguide.css.scss since it's not part of application.css.scss; otherwise, nothing in styleguide.css.scss will be available in production.

``` ruby
./config/environments/production.rb

config.serve_static_assets = true
config.assets.precompile += %w( styleguide.css )
```
