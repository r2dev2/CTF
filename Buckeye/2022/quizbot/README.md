# Quizbot

In the quizbot challenge, we are given a discord invite link and must find the flag.

![quizbot challenge prompt](https://i.imgur.com/iovKBeE.png)


## Reconnaisance

Upon entering the server, the quizbot opens a dm and asks you to start a quiz. Doing some initial reconnaisance in the server that hosts the bot, one person seems to have roles.

![rene's roles (author and admin)](https://i.imgur.com/ZF0OtY6.png)

Since `rene` seems to be an `author` and has the `admin` role, we can probably guess that one of those roles has access to a hidden channel with the flag.

Going back into the dm that quizbot has created, we are greeted by a message.

![role assign message](https://i.imgur.com/ZbJjeZe.png)

After reacting with a frog face, I noticed that I have the role named `Post meme about it`

![my roles](https://i.imgur.com/Un4anom.png)

Looking through the bot source code, it seems that whenever there is an emoji reaction of `:emoji:` to one of its messages, it sees if there's a line which has `:emoji: rolename`. If it can find a line, it adds `rolename` role to the reacter. This is the vulnerability to exploit.

## Exploitation

We have to get quizbot to output a message with `:emoji: admin` and then react with the `:emoji:` in order to get the `admin` role. Luckily, quizbot's quiz function allows you to type whatever you want and quizbot will echo the first part of your response back.

![sample quizbot response](https://i.imgur.com/4fl2gCM.png)

All we have to do is send a response in which the first name has `\n:baby: admin\newline`. Here is my injection attack:

![my injection](https://i.imgur.com/sBrrWzY.png)

And now, we have the admin role and access to the hidden channel.

![admin channel with flag](https://i.imgur.com/oOSvBZH.png)