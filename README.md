# Devise-Omniauth-Multiple-Providers

In this tutorial we will use four Social Services 'Linkedin, Twitter, Google+, Facebook', but the code will allow you to add extra social service as per the requirement of your product.

## Step 1: Add and Install Devise Gem

Modify your Gemfile to include the Devise gem

```
gem 'devise'
```
Then update your gems with

```
bundle install
```
Install the devise gem with the generator provided

```
rails generate devise:install
```
Then create your Devise model (e.g. User) using generator

```
rails generate devise User
```

Devise by default generates some migrations, so you'll need to run the created migrations

```
rake db:migrate
```
Now restart your server to get the changes

## Step 2: Add migrations to create authentication_provider and user_authentication

### authentication_provider Migration

```
class CreateAuthenticationProviders < ActiveRecord::Migration
    def change
        create_table "authentication_providers", :force => true do |t|
            t.string   "name"
            t.datetime "created_at",                 :null => false
            t.datetime "updated_at",                 :null => false
        end
        add_index "authentication_providers", ["name"], :name => "index_name_on_authentication_providers"
        AuthenticationProvider.create(name: 'facebook')
        AuthenticationProvider.create(name: 'twitter')
        AuthenticationProvider.create(name: 'gplus')
        AuthenticationProvider.create(name: 'linkedin')
    end
  end
```

### user_authentication.rb Migration

```
class CreateUserAuthentications < ActiveRecord::Migration
    def change
        create_table "user_authentications", :force => true do |t|
            t.integer  "user_id"
            t.integer  "authentication_provider_id"
            t.string   "uid"
            t.string   "token"
            t.datetime "token_expires_at"
            t.text     "params"
            t.datetime "created_at",                 :null => false
            t.datetime "updated_at",                 :null => false
        end
        add_index "user_authentications", ["authentication_provider_id"], :name => "index_user_authentications_on_authentication_provider_id"
        add_index "user_authentications", ["user_id"], :name => "index_user_authentications_on_user_id"
    end
  end
```

Run the migrations with following command

```
rake db:migrate
```

## Step 3: Add Associations in authentication_provider and user_authentication

### authentication_provider.rb

```
has_many :users
has_many :user_authentications
```

### user_authentication.rb

```
  belongs_to :user
  belongs_to :authentication_provider
  
  serialize :params

  def self.create_from_omniauth(params, user, provider)
      token_expires_at = params['credentials']['expires_at'] ? Time.at(params['credentials']['expires_at']).to_datetime : nil
      create(
              user: user,
              authentication_provider: provider,
              uid: params['uid'],
              token: params['credentials']['token'],
              token_expires_at: token_expires_at,
              params: params,
            )
  end
```

## Step 4: Create Controller users/omniauth_callbacks_controller.rb

```
  class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
    include OmniConcern
    %w[facebook twitter gplus linkedin].each do |meth|
      define_method(meth) do
        create
      end
    end
  end
```
### Add route for users/omniauth_callbacks_controller.rb in routes.rb

```
  devise_for :users, controllers: {omniauth_callbacks: 'users/omniauth_callbacks'}
```

## Step 5: Create Controller Concern omni_concern.rb

```
  module OmniConcern
    extend ActiveSupport::Concern
      def create
        auth_params = request.env["omniauth.auth"]
        provider = AuthenticationProvider.get_provider_name(auth_params.try(:provider)).first
        authentication = provider.user_authentications.where(uid: auth_params.uid).first
        existing_user = User.where('email = ?', auth_params['info']['email']).try(:first)
        if user_signed_in?
          SocialAccount.get_provider_account(current_user.id,provider.id).first_or_create(user_id: current_user.id ,  authentication_provider_id: provider.id , token: auth_params.try(:[],"credentials").try(:[],"token") , secret: auth_params.try(:[],"credentials").try(:[],"secret"))
          redirect_to new_user_registration_url
        elsif authentication
          create_authentication_and_sign_in(auth_params, existing_user, provider)
        else
          create_user_and_authentication_and_sign_in(auth_params, provider)
        end
      end
      def sign_in_with_existing_authentication(authentication)
        sign_in_and_redirect(:user, authentication.user)
      end
      def create_authentication_and_sign_in(auth_params, user, provider)
        UserAuthentication.create_from_omniauth(auth_params, user, provider)
        sign_in_and_redirect(:user, user) 
      end
      def create_user_and_authentication_and_sign_in(auth_params, provider)
        user = User.create_from_omniauth(auth_params)
        if user.valid?
            create_authentication_and_sign_in(auth_params, user, provider)
        else
            flash[:error] = user.errors.full_messages.first
            redirect_to new_user_registration_url
      end
   end
  end
```

## Step 6: Add following code to user.rb

```
  include OmniauthAttributesConcern
    
  has_many :user_authentications

  devise :omniauthable, :database_authenticatable, :registerable,:recoverable, :rememberable, :trackable

  def self.create_from_omniauth(params)
      self.send(params.provider,params)
  end
```
## Step 7: Add Model Concern omniauth_attributes_concern.rb

```
  module OmniauthAttributesConcern
      extend ActiveSupport::Concern
      module ClassMethods
          Add Methods here 
      end
  end
```
In this concern we can create Methods for each social media to fetch and store attributes

```
  def twitter params
      (params['info']['email'] = "dummy#{SecureRandom.hex(10)}@dummy.com") if params['info']['email'].blank?
      attributes = {
                      email: params['info']['email'],
                      first_name: params['info']['name'].split(' ').first,
                      last_name: params['info']['name'].split(' ').last,
                      username: params['info']['nickname'],
                      password: Devise.friendly_token
                  }
      create(attributes)
  end
```
### Note Twitter only return email address if the user has confirmed his/her email at twitter otherwise nil value is returned.

* We can add other social media accounts the same way we have added above
* Profile Image from social account can also be fetched and can be passed as

```
  remote_image_url: params['info']['image']
```

### Note the above example is meant for carrierwave gem and 'image' in remote_image_url is the DB column. You can use any other gem and pass params['info']['image'] to it.

## Step 8: Add Social Media Account Keys in devise.rb

### For Facebook

```
  config.omniauth :facebook, facebook_app_id , facebook_secret_key , :display => "popup" , :scope => 'email,publish_actions', info_fields: 'email,name'
```

### For Twitter

```
  config.omniauth :twitter, twitter_app_id , twitter_secret_key , :display => "popup" , :scope => 'email'
```

### For Linkedin

```
  config.omniauth :linkedin, linkedin_app_id ,linkedin_secret_key , :display => "popup", :scope => 'r_emailaddress,r_basicprofile'
```

### For Google+

```
  config.omniauth :gplus , gplus_app_id ,gplus_secret_key , :display => "popup" , scope: 'userinfo.email, userinfo.profile'
```

### Note: display: "popup" attribute is used when we want social media signup to open in a separate browser window

## Final Step 9: Add Gems in Gemfile for Omniauth

```
  gem 'omniauth-oauth2' , '~> 1.3.1'
  gem 'omniauth'
  gem 'omniauth-facebook'
  gem 'omniauth-twitter'
  gem 'omniauth-gplus'
  gem 'omniauth-linkedin'
```
Run the bundle command, restart the server and Vola! 










