# "Chatbots As A Service" con Messenger Platform

## Introducción

**Messenger Platform** es la plataforma de Facebook Messenger que ayuda a potenciar a las compañias a través de experiencias para el usuario a la medida como puede ser el potencial de un chatbot, pero qué sucede cuando necesitamos más de uno que realice las mismas funcionalidades pero con diferentes datos, en este tutorial aprenderemos como se conforma la estructura base de un chatbot con estas características y como debe hacerse la conexión a través de la API de Facebook para convertirlo en un servicio.

## Secciones
1. Características de un "Chatbot As A Service"
2. Estructura de datos para conectar nuestro chatbot
3. Ejemplo de un "Chatbot As A Service"
4. Uso de Open Graph API de facebook para conectar o suscribir chatbots con diferentes páginas
5. Conclusiones

## 1. Caracteristicas de un "Chatbot As A Service"

Un chatbot como servicio representa tener una base de funcionalidades que se ofrecerán a diferentes usuarios con el mismo objetivo pero desde un proveedor diferente algo común en los negocios B2B ("Business to Business").

Actualmente hay muchas plataformas que ofrecen está funcionalidad a nivel de usuario como lo son:

1. **[Chatfuel](https://chatfuel.com/)**
2.  **[ManyChat](https://manychat.com/)**

Solo por decir algunas, que además de ofrecer un servicio para conectar el chatbot a tu página de facebook de manera sencilla, ofrecen una interfaz gráfica bastante agradable para construir tus propios flujos de chatbot.

El caso de uso que explicaremos el día de hoy funciona perfectamente cuando el proveedor de los servicios eres tú y necesitas brindar tu **Super** Chatbot a más empresas o clientes pero todo dentro de tu infraestructura.

---

## 2. Estructura de datos para conectar nuestro chatbot

A continuación vamos a mostrar los datos que deberías contemplar en tu base de datos, cabe resaltar que esto es una propuesta de los datos elementales pero dependiendo de tu negocio tu podrías tener más datos que consideres necesarios.

* USER_BUSINESS: Debes considerar tener una entidad que represente al usuario que suscribe su página(s) de facebook para conectar su(s) chatbot(s), este usuario será el encargado de subir el contenido de que deberán mostrar su(s) chatbot(s).
    - User ID
    - Password
    - Access token (access_token del usuario administrador de las páginas de facebook)
    - Permissions (permisos que el usuario concede para el uso de su access_token)
  > Muy importante a considerar es que como necesitaremos tener acceso a las páginas del usuario necesitaremos guardar su **access_token** de **usuario** de facebook este lo obtendremos usando el **login de facebook** en nuestra aplicación.
* PAGE: Esta entidad almacenará los datos de cada página que suscriben los USER_BUSINESS
    - Page ID
    - Access Token (access_token de la página en particular de facebook)
    - Is active (este nos sirve para guardar el estado del chatbot)
* USER_ID: Esta entidad almacenará los datos de cada usuario que interactúa con los chatbots
    - User ID
    - State
    - Data (este nos sirve para guardar el estado de la información que proporciona el usuario al chatbot)
  >Es recomendable usar un **estado** para saber en donde se quedó el usuario en su interacción con el chatbot, sin embargo las buenas prácticas de facebook nos indican que el usuario debería poder usar el chatbot sin un estado en específico.
> MUY IMPORTANTE Y RECOMENDABLE CIFRAR EN BASE DE DATOS LOS ACCESS_TOKEN

---

## 3. Ejemplo de un "Chatbot As A Service"

En esta ocasión propondremos una estructura en la cual solo usaremos un servidor para este ***chatbot as a service*** con el objetivo de mostrar como se debe hacer la integración con Facebook, pero dependiendo del tipo de negocio tal vez deberás hacer una arquitectura más robusta.

![Diagram chatbot!](https://res.cloudinary.com/helpo/image/upload/v1603674581/tutorial/chatbot_diagram.png "Diagram chatbot")

En la imagen anterior mostramos las partes más elementales para poder cumplir con las funcionalidad que proponemos.

1. Una vez que recibimos en el webhook el mensaje proveniente del usuario en messenger es súper importante rescatar User ID y Page ID ya que estos son los identificadores de que página proviene y el identificador del usuario correspondientemente.

2. Actualmente en nuestro ejemplo generamos una instancia del chatbot en la cual tenemos las funciones necesarias para responder con las plantillas actuales de facebook messenger.

3. Tenemos una capa en la cual con base en el Page ID podemos obtener del backend los datos que se deben responder para los usuarios de esta página, por ejemplo un chatbot de restaurantes en este caso debería obtener la información del restaurante en particular del cual están llegando los mensajes y responder con el menú correspondiente.

4. Antes de responder obtendremos de nuestro backend el access token de la página de facebook en cuestión con el cual tenemos que responder al usuario que hace la consulta.[1]

5. Los datos con los cuales responderemos a la página indicada de la cual el usuario hizo la consulta lo haremos con el User ID y el access token de la página correspondiente.

> [1]. Tomar en cuenta la estructura del punto 2 de este tutorial para los datos que estamos usando, dado que **nuestro ejemplo maneja de manera independiente los servicios de base de datos y no están incluidos en el código**.

A continuación agregaremos el código ejemplo de nuestro chatbot base para que puedas consultar como es que en nuestro caso manejamos los datos

```JS
// src/bot/BotTemplate.js

const MiddlePayload = require('../src/custom_modules/MiddlePayload.js')
const ChatbotServices = require('../src/api/server')

const StatesMachine = require('./StatesMachine')
const Bot = require('../src/jsmessenger/Bot')

const BotTemplate = {}

const loadState = (id) => {
  return ChatbotServices.getState(id).then(state => state)
}

const loadPageAccess = (pageId) => {
  return ChatbotServices.getPageAccessToken(pageId).then(state => state)
}

BotTemplate.listen = async (message) => {
  // GET User ID
  const userId = message.sender.id
  // GET Page ID
  const pageId = message.recipient.id
  // GET Chatbot user state saved in DB
  const state = await loadState(userId)
  // GET Access token page saved in DB
  const accessPage = await loadPageAccess(pageId)
  // Create instance bot with access token and page id
  const bot = new Bot(accessPage, pageId)
  const stateMachine = new StatesMachine(bot, userId, pageId, ChatbotServices)
  // Bot actions input userID, this bot instance includes access token and page id yet
  bot.senderAction(userId, "mark_seen").then(res => {
    bot.senderAction(userId, "typing_on").then(res => {
       // State machine derive the next state in chatbot
      stateMachine.set(pageId, state)
      stateMachine.go(state, MiddlePayload.filter(message))
    })
  })
}

module.exports = BotTemplate;
```
[Github chatbot example code](https://github.com/JoseAngelChepo/chatbot-as-a-service)

---

## 4. Uso de Open Graph API de facebook para conectar o suscribir chatbots con diferentes páginas

En este paso veremos como obtener las páginas de los usuarios y como hacer la conexión de estas.

> Usaremos el sdk facebook para todos los ejemplos que se exponen a continuación.

```JS
export function loadFbSdk () {
  return new Promise(resolve => {
    window.fbAsyncInit = function () { // eslint-disable-line func-names
      FB.init({
        appId: process.env.APP_ID,
        xfbml: false,
        version: 'v8.0',
        cookie: true
      })
      FB.AppEvents.logPageView()
      resolve('SDK Loaded!')
    };
    (function (d, s, id) { // eslint-disable-line func-names
      const fjs = d.getElementsByTagName(s)[0]
      if (d.getElementById(id)) { return }
      const js = d.createElement(s); js.id = id
      js.src = '//connect.facebook.net/en_US/sdk.js'
      fjs.parentNode.insertBefore(js, fjs)
    }(document, 'script', 'facebook-jssdk'))
  })
}
```

**Obtener páginas**
En este ejemplo hacemos la consulta de la lista de páginas que administra el usuario,la lista incluye el **page_id** de cada página.
```JS
export function getPages (accessTokenUser) {
  return new Promise(resolve => {
    window.FB.api('me/accounts', 'GET', {'fields': 'picture, access_token, name', 'access_token': accessTokenUser}, response => resolve(response))
  })
}
```
**Conectar páginas**

A continuación expondremos dos casos bastantes importantes para conectar y desconectar una página del chatbot y mostraremos como se aplican para la API de Facebook, ambos ejemplos a continuación se usaron en el front-end de la aplicación.

![Diagram services!](https://res.cloudinary.com/helpo/image/upload/v1603685312/tutorial/services.png "Diagram services")

* **Conectar Página**: el conectar una página para poder usar el chatbot en ella consta de 3 pasos elementales.

1. Extend token: en este caso se le otorgan los permisos al token de la página para poder hacer la suscripción.

  * Se usan los siguientes parámetros:

    - accessTokenPage: token de la página que obtenemos de la lista de páginas del usuario.
    - clientSecret: este parámetro debe ser de la aplicación que tu creaste para el chatbot base.
    - clientId: este parámetro debe ser de la aplicación que tu creaste para chatbot base.


```JS
export function extendToken (accessTokenPage) {
  return new Promise(resolve => {
    const clientSecret = process.env.CLIENT_SECRET
    const clientId = process.env.APP_ID
    window.FB.api('/oauth/access_token', 'GET', {'fb_exchange_token': accessTokenPage, 'client_secret': clientSecret, 'client_id': clientId, 'grant_type': 'fb_exchange_token'}, response => resolve(response))
  })
}
```
2. Se hace la suscripción de la página a la aplicación de facebook que "hostea" al chatbot base.
  * Se usan los siguientes parámetros:
    - accessTokenPage: token que nos regresa extendToken en el paso anterior.
    - pageId: identificador de la página que obtenemos de la lista de páginas del usuario.

```JS
export function subscribePage (pageId, accessTokenPage) {
  // YOU SHOULD SAVE THIS PAGE_ID AND ACCESS_TOKEN IN YOUR BACK_END APP TO SHARE WITH CHATBOT BASE
  return new Promise(resolve => {
    window.FB.api(pageId + '/subscribed_apps', 'POST', {'access_token': accessTokenPage}, response => resolve(response))
  })
}
```
3. Este paso no es elemental pero si cuentas con una aplicación de wit.ai entrenada para tu chatbot y quieres heredar la funcionalidad en la página nueva, debes usar este servicio.
  * Se usan los siguientes parámetros:
    - accessTokenPage: token que nos regresa extendToken en el paso anterior.
    - customToken: token de la aplicación de wit.ai previamente creada y entrenada.

```JS
export function setNLP (accessTokenPage) {
  return new Promise(resolve => {
    const customToken = '*********************'
    window.FB.api('me/nlp_configs', 'POST', {'access_token': accessTokenPage, 'nlp_enabled': true, 'custom_token': customToken}, response => resolve(response))
  })
}
```
>Con estos tres pasos ya tendrás suscrita la página al chatbot base, pero será muy **importante guardar los datos en tu back-end** para poder compartirlos con el chatbot.

* **Desconectar Página**: para desconectar una página se deben realizar los siguientes pasos para garantizar la desuscripción.

1. Desuscribir: en este paso se hace un delete en la suscripción.

  * Se usan los siguientes parámetros:
    - accessTokenPage: token que nos regresa extendToken en el paso anterior.
    - pageId: identificador de la página que obtenemos de la lista de páginas del usuario.


```JS
export function unsubscribePage (pageId, accessTokenPage) {
  return new Promise(resolve => {
    window.FB.api(pageId + '/subscribed_apps', 'DELETE', {'access_token': accessTokenPage}, response => resolve(response))
  })
}
```
2. Delete properties: este paso se ejecuta en caso de que se haya configurado el menu persistente del chatbot, si este no fuera el caso este paso no es necesario.

  * Se usan los siguientes parámetros:
    - accessTokenPage: token que nos regresa extendToken en el paso anterior.

```JS
export function deleteProperties (accessTokenPage) {
  return new Promise(resolve => {
    window.FB.api('me/messenger_profile', 'DELETE', {'access_token': accessTokenPage, 'fields': ['persistent_menu', 'get_started', 'greeting']}, response => resolve(response))
  })
}
```
---
