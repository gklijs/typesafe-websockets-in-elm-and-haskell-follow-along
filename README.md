# typesafe-websockets-in-elm-and-haskell-follow-along
Following along with this article: https://www.paramander.com/blog/playing-with-websockets-in-haskell-and-elm Very interesting stuff!

Introduction

In this article we are going to build a type-safe realtime web application with WebSockets. Using Haskell to run the server-side and Elm for the client-application. For now we will keep it small and just have the clients send a predefined text message through the server to other connected clients.

For those who do not want to code along with this article, the source code is available on GitLab: https://gitlab.com/paramander/typesafe-websockets.

Prerequisites

Before we begin we should make sure we are working with the same tools to avoid running into trouble while coding along. At the time I wrote this article I used the following tools to run the server and client:

stack (v1.1.2)
elm (v0.17.0)
Both tools installed with matching versions? Great! Let's start by building the server.

Creating the Haskell application

Open up your terminal and navigate to where you think this project fits well on your computer. Using Stack we should first create a bare Haskell application:

> stack new typesafe-websockets simple --resolver lts-6.10
> cd typesafe-websockets
And of course we are not going to code everything from scratch so I selected some dependencies to help us out a bit. Add them to the executable.build-depends section of ./typesafe-websockets.cabal:

executable typesafe-websockets
  ..
  build-depends:       base >= 4.7 && < 5
                     , text
                     , wai
                     , wai-websockets
                     , warp
                     , websockets
                     , http-types
                     , safe
Now install those dependencies so we can actually use them in our project by running stack install --dependencies-only*.

*If this is your first time using Stack you will need to run some other commands first to get your system up and running. Just follow the suggestions Stack gives you and you should be fine.

Setting up the server

Let's open ./src/Main.hs in your favourite editor and start coding! This file should start with the Language Pragma's we like to use, the module definition and the import-statements for the dependencies. After adding all those your file should look similar to:

{-# LANGUAGE OverloadedStrings   #-}
{-# LANGUAGE ScopedTypeVariables #-}

module Main where

import qualified Control.Concurrent             as Concurrent
import qualified Control.Exception              as Exception
import qualified Control.Monad                  as Monad
import qualified Data.List                      as List
import qualified Data.Maybe                     as Maybe
import qualified Data.Text                      as Text
import qualified Network.HTTP.Types             as Http
import qualified Network.Wai                    as Wai
import qualified Network.Wai.Handler.Warp       as Warp
import qualified Network.Wai.Handler.WebSockets as WS
import qualified Network.WebSockets             as WS
import qualified Safe

main :: IO ()
main = do
  putStrLn "hello world"
The main-function of our application should take care of running the server. First we create a new MVar to keep our applications state, which we will discus in more detail later. We are going to use Warp to run our web application by calling run with 2 arguments, the port to run the application on (3000) and the application itself. The application decides, based on the request, whether to respond with the WebSocket application or a HTTP application. We will look at the implementation of the WebSocket application later on, for now we pass it the reference to the initial state and let Warp run it with the default options. Our HTTP application is going to return a 400 status and inform the visitor that we expect a WebSocket request, in a follow-up article we will have the HTTP application serve the client-application.

main :: IO ()
main = do
  state <- Concurrent.newMVar []
  Warp.run 3000 $ WS.websocketsOr
    WS.defaultConnectionOptions
    (wsApp state)
    httpApp

httpApp :: Wai.Application
httpApp _ respond = respond $ Wai.responseLBS Http.status400 [] "Not a websocket request"
Adding server helpers

Next we are going to define the custom types our application is going to need to do it's job and some helpers to work with them.

Multiple clients should be able to connect to our server and we should know how to reach each of them. So we would need an unique identifier for each client, for now let's use a simple Int for this. With this identifier we need to be able to find the associated connection, because this is all the information we need at this moment we can just use a tuple containing the identifier and the connection (WS.Connection). Our final application state should then be a list of those tuples, for which we already created an initial value (an empty List) in our main-function.

type ClientId = Int
type Client   = (ClientId, WS.Connection)
type State    = [Client]
Before we can add a new client to the application state we should pick a unique id for the client. Let's add a function called nextId which takes a state, find the highest id currently in there and increments it by 1 or if it's the first connection returns 0.

nextId :: State -> ClientId
nextId = Maybe.maybe 0 ((+) 1) . Safe.maximumMay . List.map fst
Now we have everything we need to add new clients to our state so we should create a new function. This function takes the clients connection and the reference to the current state and finally returns the id for the added client. We call modifyMVar which safely updates the state and inside it's handler we use nextId on the current state to generate the clients identifier. Then we add the new Client to the list and return this new state.

connectClient :: WS.Connection -> Concurrent.MVar State -> IO ClientId
connectClient conn stateRef = Concurrent.modifyMVar stateRef $ \state -> do
  let clientId = nextId state
  return ((clientId, conn) : state, clientId)
But now what if the clients connection is lost? Good question, let's make sure to remove the client from the state when this happens. Because we gave each client an id we could use this to filter the state, removing the client we want out. This callback-function should update the state just like we did when adding clients. Because we do not need to return anything this time we can use the silent version of modifyMVar by appending a _ to it.

withoutClient :: ClientId -> State -> State
withoutClient clientId = List.filter ((/=) clientId . fst)

disconnectClient :: ClientId -> Concurrent.MVar State -> IO ()
disconnectClient clientId stateRef = Concurrent.modifyMVar_ stateRef $ \state ->
  return $ withoutClient clientId state
Writing the core part of our server

We want a listener which waits forever until the client sends a message and then broadcast that message to all other clients and repeat. We should pass this function a connection to listen on, the clients id and a reference to all other clients (the state). When a message is received we read the current state remove the sender (luckily we already wrote withoutClient as a separate function) and loop over all other clients and send them the message.

listen :: WS.Connection -> ClientId -> Concurrent.MVar State -> IO ()
listen conn clientId stateRef = Monad.forever $ do
  WS.receiveData conn >>= broadcast clientId stateRef

broadcast :: ClientId -> Concurrent.MVar State -> Text.Text -> IO ()
broadcast clientId stateRef msg = do
  clients <- Concurrent.readMVar stateRef
  let otherClients = withoutClient clientId clients
  Monad.forM_ otherClients $ \(_, conn) ->
    WS.sendTextData conn msg
Having all parts of our WebSocket server available now we need to wire it all together. The server receives a pending connection, which we should accept, add it to the state and then start listening until the client disconnects. To make sure connections are kept alive we also fork a ping thread which simply pings the client every 30 seconds.

wsApp :: Concurrent.MVar State -> WS.ServerApp
wsApp stateRef pendingConn = do
  conn <- WS.acceptRequest pendingConn
  clientId <- connectClient conn stateRef
  WS.forkPingThread conn 30
  Exception.finally
    (listen conn clientId stateRef)
    (disconnectClient clientId stateRef)
That's all there is to the server for now, let's continue on the Elm client application.

Building the Elm client

The client application will remain very simple for this article and only requires 2 dependencies, so let's install them:

> elm package install elm-lang/html -y
> elm package install elm-lang/websocket -y
Next up, change the source in your elm-package.json file that was just created by the install commands above:

{
  ...
  "source-directories": [
    "src"
  ],
  ...
}
Then we create ./src/Main.elm and add the module-definition, imports and the main-function initialising a basic html program:

module Main exposing (main)

import Html.App    as App
import Html        exposing (..)
import Html.Events exposing (..)
import WebSocket

main : Program Never
main =
  App.program
     { init          = init
     , update        = update
     , view          = view
     , subscriptions = subscriptions
     }
All we are going to do is keep count of all the "poke"'s we receive from the server and make it possible to send them as well. So our Model can be a simple Int and the messages we need to handle are Receive String and Send.

type alias Model
  = Int

type Msg
  = Receive String
  | Send
The initial model for the client should be 0 as we did not receive anything yet. The view should display the current amount of pokes received and a button which triggers a Send within Elm.

init : (Model, Cmd Msg)
init =
  (0, Cmd.none)

view : Model -> Html Msg
view model =
  div []
    [ p [] [ text <| "Pokes: " ++ toString model ]
    , button [ onClick Send ] [ text "Poke others" ]
    ]
We are almost there! We just need to implement update and subscribe to messages from the server. Make sure you check the data received from the server because we only want to increment the model on "poke"'s and just ignore everything else. Sending and listening to the server is easily done with the send and listen functions provided by elm-lang/websocket.

wsUrl : String
wsUrl = "ws://localhost:3000"

update : Msg -> Model -> (Model, Cmd Msg)
update msg model =
  case msg of
    Receive "poke" ->
      (model + 1) ! []
    Receive _ ->
      model ! []
    Send ->
      model ! [ WebSocket.send wsUrl "poke" ]

subscriptions : Model -> Sub Msg
subscriptions model =
  WebSocket.listen wsUrl Receive
Seeing server and client in action

That's it, we created a WebSocket server and client in Haskell and Elm. Now it's time to see both work! In your terminal start the server:

> stack runhaskell ./src/Main.hs
And in another window the client:

> elm reactor
Navigate at least 2 browser windows to http://localhost:8000/src/Main.elm and start poking!

What's next?
