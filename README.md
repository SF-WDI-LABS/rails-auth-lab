#Simple Authentication with Bcrypt

This tutorial is for adding authentication to a vanilla Ruby on Rails app using Bcrypt and has\_secure\_password. 

The steps below are based on Ryan Bates's approach from [
Railscast \#250 Authentication from Scratch (revised)](http://railscasts.com/episodes/250-authentication-from-scratch-revised). 

You can see the final source code here: [repo](https://github.com/thebucknerlife/gif_vault). I began with a stock rails app using ``` rails new gif_vault ```

##Steps

1. Create a user model with a name, email and password\_digest (all strings) by entering the following command into the command line: ``` rails generate model user name email password_digest ```. 

    *Note:* If you already have a user model or you're going to use a different model for authentication, that model must have an attribute names password\_digest and some kind of attribute to identify the user (like an email or a username).
2. Run ``` rake db:migrate ``` in the command line to migrate the database.
3. Add these routes below to your routes.rb file. Notice I also deleted all the comments inside that file. Don't forget to leave the trailing ``` end ```, though.
    
    ```ruby
    # config/routes.rb

    GifVault::Application.routes.draw do
        
        # This route sends requests to our naked url to the *cool* action in the *gif* controller.
        root to: 'gif#cool'
        
        # I've created a gif controller so I have a page I can secure later. 
        # This is optional (as is the root to: above).
        get '/cool' => 'gif#cool'
        get '/sweet' => 'gif#sweet'

        # These routes will be for signup. The first renders a form in the browse, the second will 
        # receive the form and create a user in our database using the data given to us by the user.
        get '/signup' => 'users#new'
        post '/users' => 'users#create'

    end
    ```
    
3. Create a users controller: 

    ```ruby
    # app/controllers/users_controller.rb
    
    class UsersController < ApplicationController

    end
    ```
    
4. Add a **new** action (for rendering the signup form) and a **create** action (for receiving the form and creating a user with the form's parameters.): 

    ```ruby
    # app/controllers/users_controller.rb
    
    class UsersController < ApplicationController

        def new
        end

        def create
        end   

    end
    ```
    
5. Now create the view file where we put the signup form. 

    ```html+erb
    <!-- app/views/users/new.html.erb -->
    
    <h1>Signup!</h1>

    <%= form_for :user, url: '/users' do |f| %>

      Name: <%= f.text_field :name %>
      Email: <%= f.text_field :email %>
      Password: <%= f.password_field :password %>
      Password Confirmation: <%= f.password_field :password_confirmation %>
      <%= f.submit "Submit" %>

    <% end %>
    ```
   *A note on Rail's conventions:* This view file is for the **new** action of the **users controller**. As a result, we save the file here: ``` /app/views/users/new.html.erb ```. The file is called **new**.html.erb and it is saved inside the views folder, in a folder we created called **users**. 
   
   That's the convention: view files are inside a folder with the same name as the controller and are named for the action they render.
    
6. Add logic to **create** action and add the private ``` user_params ``` method to sanitize the input from the form (this is a new Rails 4 thing and it's required). You might need to adjust the parameters inside the ``` .permit() ``` method based on how you setup your User model.

    ```ruby
  class UsersController < ApplicationController

    def new
    end

    def create
      user = User.new(user_params)
      if user.save
        session[:user_id] = user.id
        redirect_to '/'
      else
        redirect_to '/signup'
      end
    end

  private

    def user_params
      params.require(:user).permit(:name, :email, :password, :password_confirmation)
    end
  end
    ```

7. Go to your Gemfile and uncomment the 'bcrypt' gem. We need bcrypt to securely store passwords in our database. 

    ```ruby
    source 'https://rubygems.org'

    # Bundle edge Rails instead: gem 'rails', github: 'rails/rails'
    gem 'rails', '4.0.4'

    # Use sqlite3 as the database for Active Record
    gem 'sqlite3'

    ...

    # Use ActiveModel has_secure_password
    gem 'bcrypt', '~> 3.1.7'

    ...
    
    ```
    
7. Go to the User model file and add ``` has_secure_password ```. This is the line of code that gives our User model authentication methods via bcrypt.

    ```ruby
    # app/models/user.rb
        
    class User < ActiveRecord::Base

      has_secure_password

    end
    ```
    
9. Run ``` bundle install ``` from the terminal then restart your rails server.

   *Note:* Windows users might have issues with bcrypt. If so, copy the error into Google and look for answers on Stack Overflow. There is documentation online for how to fix Windows so the bcrypt works.
   
10. Create a sessions controller. This is where we create (aka login) and destroy (aka logout) sessions.

    ```ruby
    # app/controllers/sessions_controller.rb
    
    class SessionsController < ApplicationController

      def new
      end

      def create
      end

      def destroy
      end

    end
    ```
    
11. Create a form for user's to login with.

    ```html+erb
    <!-- app/views/sessions/new.html.erb -->
    
    <h1>Login</h1>

    <%= form_tag '/login' do %>

      Email: <%= text_field_tag :email %>
      Password: <%= password_field_tag :password %>
      <%= submit_tag "Submit" %>

    <% end %>
    ```
    
12. Update your routes file to include new routes for the sessions controller.

    ```ruby
    GifVault::Application.routes.draw do

      root to: 'gif#cool'

      # these routes are for showing users a login form, logging them in, and logging them out.
      get '/login' => 'sessions#new'
      post '/login' => 'sessions#create'
      get '/logout' => 'sessions#destroy'

      get '/signup' => 'users#new'
      post '/users' => 'users#create'
      
    end
    ```

13. Update the sessions_controller with the logic to log users in and out.

    ```ruby
      # app/controllers/sessions_controller.rb
    
      def create
        user = User.find_by_email(params[:email])
        # If the user exists AND the password entered is correct.
        if user && user.authenticate(params[:password])
          # Save the user id inside the browser cookie. This is how we keep the user 
          # logged in when they navigate around our website.
          session[:user_id] = user.id
          redirect_to '/'
        else
        # If user's login doesn't work, send them back to the login form.
          redirect_to '/login'
        end
      end

      def destroy
        session[:user_id] = nil
        redirect_to '/login'
      end
    ```
    
14. Update the application controller with new methods to look up the user, if they're logged in, and save their user object to a variable called @current\_user. The ``` helper_method ``` line below current\_user allows us to use ``` @current_user ``` in our view files. Authorize is for sending someone to the login page if they aren't logged in - this is how we keep certain pages our site secure... user's have to login before seeing them.

    ```ruby
    # app/controllers/application_controller.rb
    
    class ApplicationController < ActionController::Base
      # Prevent CSRF attacks by raising an exception.
      # For APIs, you may want to use :null_session instead.
      protect_from_forgery with: :exception

      def current_user
        @current_user ||= User.find(session[:user_id]) if session[:user_id]
      end
      helper_method :current_user

      def authorize
        redirect_to '/login' unless current_user
      end

    end
    ```

15. Add a ``` before_filter ``` to any controller that you want to secure. This will force user's to login before they can see the actions in this controller. I've created a gif controller below which I'm going to secure. The routes for this controller were added to the routes.rb in the beginning of this tutorial.

    ```ruby
    # app/controllers/gif_controller.rb

    class GifController < ApplicationController

      before_filter :authorize

      def cool
      end

      def free
      end

    end
    ```

16. You can update your application layout file to show the user's name if they're logged in and some contextual links.

    ```html+erb
    <!-- app/views/layout/application.html.erb -->
    
    <!DOCTYPE html>
    <html>
    <head>
      <title>GifVault</title>
      <%= stylesheet_link_tag    "application", media: "all", "data-turbolinks-track" => true %>
      <%= javascript_include_tag "application", "data-turbolinks-track" => true %>
      <%= csrf_meta_tags %>
    </head>
    <body>

    # added these lines.
    <% if current_user %>
      Signed in as <%= current_user.name %> | <%= link_to "Logout", '/logout' %>
    <% else %>
      <%= link_to 'Login', '/login' %> | <%= link_to 'Signup', '/signup' %>
    <% end %>

    <%= yield %>

    </body>
    </html>
    ```

## Things Missing

* Adding validations (this afternoon)

--
All done! Feel free to fork and update this. Reach me at @thebucknerlife on Twitter.
