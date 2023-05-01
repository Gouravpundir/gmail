# gmail

```
const { google } = require('googleapis');
const { OAuth2 } = google.auth;
require('dotenv').config();
const oauth2Client = new OAuth2(
  process.env.CLIENT_ID,
  process.env.CLIENT_SECRET,
  process.env.REDIRECT_URI
);

oauth2Client.setCredentials({
  refresh_token: process.env.REFRESH_TOKEN
});

const gmail = google.gmail({
  version: 'v1',
  auth: oauth2Client
});

// Function to check for new emails in Gmail
const checkNewEmails = async () => {
  const res = await gmail.users.messages.list({
    userId: 'me',
    q: 'is:unread'
  });

  const messages = res.data.messages || [];

  messages.forEach(async (msg) => {
    const message = await gmail.users.messages.get({
      userId: 'me',
      id: msg.id
    });

    const threadId = message.data.threadId;

    const headers = message.data.payload.headers;

    let isReply = false;

    headers.forEach((header) => {
      if (header.name === 'From' && header.value.includes('youremail@gmail.com')) {
        isReply = true;
      }
    });

    if (!isReply) {
      await sendReply(message, threadId);
    }
  });
};

// Function to send reply to an email and label it
const sendReply = async (message, threadId) => {
  const subject = message.data.payload.headers.find(header => header.name === 'Subject').value;

  const reply = `Hi, thank you for your email on the subject of ${subject}. I'm currently out of the office, but I'll get back to you as soon as possible.`;

  await gmail.users.messages.send({
    userId: 'me',
    requestBody: {
      threadId: threadId,
      message: {
        raw: Buffer.from(
          `To: ${message.data.payload.headers.find(header => header.name === 'From').value}\r\n` +
          `Subject: Re: ${subject}\r\n` +
          '\r\n' +
          `${reply}`
        ).toString('base64')
      }
    }
  });

  const label = await gmail.users.labels.create({
    userId: 'me',
    requestBody: {
      name: 'Auto-replied'
    }
  });

  await gmail.users.messages.modify({
    userId: 'me',
    id: message.data.id,
    requestBody: {
      addLabelIds: [label.data.id],
      removeLabelIds: ['INBOX']
    }
  });
};

// Function to run the app in random intervals of 45 to 120 seconds
const run = async () => {
  try {
    await checkNewEmails();

    const randomInterval = Math.floor(Math.random() * (120 - 45 + 1) + 45);

    setTimeout(run, randomInterval * 1000);
  } catch (error) {
    console.log(error);
  }
};

run();

```
