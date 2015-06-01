---
layout: post
title: "APIs with Rails:<br/>render :json => the_simple_way"
date: 2015-05-23 14:50:41 +0200
comments: true
categories: ruby rails api json
author: Alessandro Verlato (@Aleverla)
published: true
---

Negli ultimi tempi il mio lavoro in Fancy Pixel si è focalizzato sulla realizzazione del backend di un prodotto 
che stiamo per lanciare (link Space Bunny?) e per il quale abbiamo deciso di realizzare un backend esclusivamente ad API JSON, utilizzate da terzi ma anche dal nostro frontend. In questo breve articolo vorrei condividere con voi la semplice soluzione che stiamo utilizzando per la generazione del JSON di risposta e che a mio avviso può essere un semplice trucchetto alternativo ai sistemi più utilizzati come Jbuilder e ActiveModel::Serializers.

<!-- More -->

Parliamo di Rails, uno dei miei compagni di lavoro preferiti e con il quale mi è capitato parecchie volte di lavorare su progetti che prevedevano lo sviluppo di API. Nel corso degli anni ho avuto occasione di provare per curiosità personale diverse soluzioni per la generazione del JSON di risposta e devo dire che a volte ho trovato delle difficoltà con alcune tecnologie, bellissime per la maggior parte delle loro funzionalità, ma che ti costringono a fare giri strani e forzature in alcuni casi particolari. A dirla tutta probabilmente il motivo principale di questa continua evoluzione è la costante ricerca del miglior bilanciamento fra comodità/facilità/velocità di sviluppo e performance e dopo tutto questo provare, sono arrivato alla soluzione attuale, che probabilmente non è nulla di nuovo, ma che magari a qualcuno può fornire un'idea alternativa che secondo me riesce a combinare insieme grande flessibilità e performance veramente vicine al metallo nudo.  

## To cut a long story short... 

Non mi voglio dilungare in racconti epici o disquisizioni sul sesso degli Angeli (offensivo per some 'MMMuricans?) e quindi ho preparato una banale [applicazione demo](https://github.com/FancyPixel/serializers_demo), che potete trovare 
sul nostro account GitHub, con la quale mostrare velocemente di cosa sto parlando.

Setuppiamo velocemente l'applicazione:

~~~bash
git clone "https://github.com/FancyPixel/serializers_demo"
cd serializers_demo
bundle
rake db:create && rake db:migrate && rake db:seed
~~~

Ho deciso di utilizzare [rails-api](https://github.com/rails-api/rails-api) (se non lo conoscete vi consiglio di dargli un'occhiata) al posto di Rails standard per il semplice fatto che è quello che sto utilizzando anche io nel progetto di cui vi parlavo all'inizio dell'articolo , ma ovviamente gli stessi discorsi valgono identici per Rails standard. 
Apriamo insieme il codice e diamogli un'occhiata veloce: come potete vedere queste poche righe di sorgente non fanno altro che rispondere a 3 rotte e se buttate uno sguardo al file ```config/routes.rb``` troverete qualcosa del tipo:

~~~ruby
# config/routes.rb

Rails.application.routes.draw do
  namespace :v1, defaults: { format: :json } do
    get :jbuilder, to: 'comparison#jbuilder'
    get :ams, to: 'comparison#ams'
    get :simple, to: 'comparison#simple'
  end
end 
~~~

Come vedete ho impostato ```json``` come formato di default e ho definito un namespace per replicare un tipico modo di versionare un API.
Passiamo velocemente all'unico controller presente (```comparison_controller```) dove troviamo l'implementazione delle action richiamate dalle routes. Ognuna di queste fa esattamente la stessa cosa: carica una serie di record dal DB e li renderizza come JSON, ma ognuna lo fa "a modo suo", cioè utilizzando rispettivamente jbuilder, ActiveModel::Serializers e la mia idea soluzione che ho chiamato "simple"... che fantasia eh?

Non ci soffermiamo sui primi due sistemi, in quanto probabilmente li conoscete meglio di me e non hanno nulla di fuori standard nella mia implementazione. Passiamo direttamente alla action ```simple```. Come per i concorrenti prima di lei, non fa altro che renderizzare del JSON, ma questa volta chiamando in aiuto il metodo ```serialize_awesome_stuffs``` definito nel modulo 

~~~ruby
V1::SimpleAwesomeStuffSerializer
~~~

e che è importato dal controller.
Trovate il modulo sotto la cartella ```app/serializers/v1``` e se lo aprite noterete che altro non è se non un normalissimo modulo che definisce dei metodi. Diamo un'occhiata insieme al contenuto

~~~ruby
# app/serializers/v1/simple_awesome_stuff_serializer.rb

module V1
  module SimpleAwesomeStuffSerializer

    def serialize_awesome_stuff(awesome_stuff = @awesome_stuff)
      {
          name: awesome_stuff.name,
          some_attribute: awesome_stuff.some_attribute,
          a_counter: awesome_stuff.a_counter
      }
    end

    def serialize_awesome_stuffs(awesome_stuffs = @awesome_stuffs)
      {
          awesome_stuffs: awesome_stuffs.map do |awesome|
            serialize_awesome_stuff awesome
          end
      }
    end
  end
end
~~~

Come vedete entrambi i metodi non fanno altro che ritornare Hash con definite le coppie chiave - valore che, inutile dirlo, vogliamo ritornare come JSON. In particolare ```serialize_awesome_stuffs``` crea un ```Array``` di ```Hash``` e internamente, per "DRYing up" chiama ```serialize_awesome_stuff``` (singular). Forse la scelta dei nomi non è delle più felici vero?  Bonus point: definendo il parametro ```awesome_stuffs = @awesome_stuffs``` ci permette di rendere ancora più 'leggero' e leggibile il nostro codice in quanto se siamo rimasti aderenti al naming convenzionale delle variabili, probabilmente il nostro controller avrà definito qualcosa come  ```@awesome_stuff``` (e avete visto che lo abbiamo fatto) che è direttamente visibile ed utilizzabile dal nostro metodo, avendo incluso il modulo nel controller. Nel caso però ci venisse un attacco di creatività e volessimo utilizzare dei nomi di variabili personalizzati, non avremmo nessun problema. Prendete ad esempio questo pezzo di codice:

~~~ruby
# some_controller.rb

def some_action
  @my_personal_awesome_stuffs = AwesomeStuff.all
  render json: serialize_awesome_stuffs @my_personal_awesome_stuffs
end
~~~

and everything will work as expected.

----- Seconda parte? ------

## Step-by-step: let's complicate things a bit 

Andiamo insieme ad aumentare un po' la complessità al nostro esempio, aggiungendo il modello ```User``` e definendo una relazione one-to-many con il nostro AwesomeStuff:

~~~bash
rails g model user name:string
~~~

Aggiungiamo la reference a User in AwesomeStuff:

~~~bash
rails g migration add_user_reference_to_awesome_stuff user:references
~~~

E migriamo tutto

~~~bash
rake db:migrate
~~~

Definiamo ora le relazioni tra i due modelli:

~~~ruby
# app/models/awesome_stuff.rb
class AwesomeStuff < ActiveRecord::Base
  belongs_to :user
end

# app/models/user.rb
class User < ActiveRecord::Base
  has_many :awesome_stuffs
end
~~~

lanciamo la console di Rails con ```rails c``` e inseriamo in DB un po' di dati di test:

~~~ruby
# Inseriamo 5 utenti
users = []
5.times {|n| users << User.create(name: "user_#{n}") }

# Associamo in modo random i nostri awesome record™ agli utenti 
AwesomeStuff.all.each { |aww| aww.update user: users.sample }

# Facciamo un test veloce
User.first.awesome_stuffs

=> #<ActiveRecord::Associations::CollectionProxy [#<AwesomeStuff id: ... ... >]>
~~~

Adesso che abbiamo preparato tutto percorriamo alcuni dei passi che faremmo normalmente durante la creazione di un API. Andiamo quindi a creare un controller per gli ```User``` attraverso il quale restituiremo al client anche tutti gli awesome records™ associati.
Creiamo quindi il file ```app/controllers/v1/users_controller.rb``` e iniziamo creando la action ```index```:

~~~ruby
# app/controllers/v1/users_controller.rb

module V1
  class UsersController < ApplicationController
    include V1::UsersSerializer

    def index
      @users = User.all.includes(:awesome_stuffs)
      render json: serialize_users
    end
  end
end
~~~

Come vedete ho già aggiunto alcune cose che ci serviranno tra poco e cioè il modulo ```V1::UsersSerializer``` che contiene i metodi di serializzazione. Se non lo avete già fatto, notate lo scoping dei nostri serializers (V1): così fancendo possiamo seguire senza problemi l'evoluzione delle versioni delle nostre API, andando eventualmente a ridefinire il comportamento solo dei serializers che dovessero cambiare.
Non dimentichiamoci di specificare la nuova route:

~~~ruby
# config/routes.rb

Rails.application.routes.draw do
  namespace :v1, defaults: { format: :json } do
    get :jbuilder, to: 'comparison#jbuilder'
    get :ams, to: 'comparison#ams'
    get :simple, to: 'comparison#simple'

    # Let's add the 'index' only
    resources :users, only: :index
  end
end
~~~

Cosa mettiamo nel nostro ```UsersSerializer``` ? Una prima idea potrebbe essere qualcosa del tipo:

~~~ruby
# app/serializers/v1/users_serializer.rb

module V1
  module UsersSerializer

    def serialize_users(users = @users)
      {
        users: users.map do |user|
          {
              id: user.id,
              name: user.name,
              awesome_stuffs: user.awesome_stuffs.map do |aww|
                {
                    name: aww.name,
                    some_attribute: aww.some_attribute,
                    a_counter: aww.a_counter
                }
              end
          }
        end
      }
    end
  end
end
~~~

C'è un po' di puzza di codice qui, non trovate? Una parte di questa roba l'abbiamo già vista, vediamo di riutilizzarla:

~~~ruby
# app/serializers/v1/users_serializer.rb

module V1
  module UsersSerializer
    include V1::SimpleAwesomeStuffSerializer

    def serialize_users(users = @users)
      {
        users: users.map do |user|
          {
              id: user.id,
              name: user.name,
              awesome_stuffs: user.awesome_stuffs.map do |aww|
                serialize_awesome_stuff aww
              end
          }
        end
      }
    end
  end
end
~~~

Molto bene, ma possiamo fare qualcosa di meglio, visto che quasi sicuramente la nostra API avrà anche una route che risponde con i dati relativi ad un utente, cioè qualcosa di simile a ```/v1/users/1``` possiamo portarci avanti con il lavoro, asciugando contemporaneamente il codice attuale:

~~~ruby
# app/serializers/v1/users_serializer.rb

module V1
  module UsersSerializer
    include V1::SimpleAwesomeStuffSerializer

    def serialize_user(user = @user)
      {
          id: user.id,
          name: user.name,
      }
    end

    def serialize_users(users = @users)
      {
        users: users.map do |user|
          serialize_user(user).merge(
            {
                awesome_stuffs: user.awesome_stuffs.map do |aww|
                  serialize_awesome_stuff aww
                end
            }
          )
        end
      }
    end
  end
end
~~~

 ---> WARNING ANIMALISTI!!! Ok, we've just killed two birds with one stone.

Avrete già notato che è possibile ottenere un ulteriore miglioramento:

~~~ruby
# app/serializers/v1/users_serializer.rb

module V1
  module UsersSerializer
    include V1::SimpleAwesomeStuffSerializer

    def serialize_user(user = @user)
      {
          id: user.id,
          name: user.name
      }
    end

    def serialize_users(users = @users)
      {
        users: users.map do |user|
          serialize_user(user).merge(serialize_awesome_stuffs(user.awesome_stuffs))
        end
      }
    end
  end
end
~~~

Siamo arrivati alla fine, spero di non avervi annoiato, ma se siete arrivati a leggere fino a qui forse è così 😊
Quello che avete visto oggi può o meno piacere, ma personalmente lo trovo un sistema che magari non è super elegante, ma è sicuramente performante e offre una grandissima modularità, estendibilità e riuso.

Feel free to leave a comment, we’d really love to hear your feedback.

See ya soon!

Alessandro - @madAle
