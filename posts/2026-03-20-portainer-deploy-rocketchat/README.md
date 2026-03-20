# How to Deploy Rocket.Chat via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Rocket.Chat, Team Chat, Collaboration, Self-Hosted

Description: Deploy Rocket.Chat via Portainer as a full-featured team communication platform with voice/video calls, live chat, and omnichannel messaging support.

## Introduction

Rocket.Chat is a feature-rich, open-source team communication platform with voice and video calling, live chat widget integration, and omnichannel support for customer service. Deploy via Portainer with MongoDB for a complete communication hub.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  rocketchat:
    image: rocketchat/rocket.chat:latest
    container_name: rocketchat
    command: >
      bash -c
        "for i in `seq 1 30`; do
          node main.js &&
          s=$$? && break || s=$$?;
          echo \"Tried $$i times. Waiting 5 secs...\";
          sleep 5;
        done; (exit $$s)"
    environment:
      MONGO_URL: mongodb://rocketchat-db:27017/rocketchat?replicaSet=rs0&directConnection=true
      MONGO_OPLOG_URL: mongodb://rocketchat-db:27017/local?replicaSet=rs0&directConnection=true
      ROOT_URL: https://chat.example.com
      PORT: 3000
      DEPLOY_PLATFORM: docker-compose
      OVERWRITE_SETTING_Show_Setup_Wizard: completed
    volumes:
      - rocketchat_data:/app/uploads
    ports:
      - "3000:3000"
    depends_on:
      - rocketchat-db
    restart: unless-stopped

  # MongoDB with Replica Set (required for Rocket.Chat)
  rocketchat-db:
    image: mongo:6.0
    container_name: rocketchat-db
    command: mongod --replSet rs0 --oplogSize 128
    volumes:
      - rocketchat_db:/data/db
    restart: unless-stopped

  # Initialize MongoDB replica set
  mongo-init-replica:
    image: mongo:6.0
    command: >
      bash -c "
        sleep 10 &&
        mongosh rocketchat-db/rocketchat --eval 'rs.initiate({_id: \"rs0\", members: [{_id: 0, host: \"rocketchat-db:27017\"}]})'
      "
    depends_on:
      - rocketchat-db

volumes:
  rocketchat_data:
  rocketchat_db:
```

## Initial Setup

1. Access `http://<host>:3000`
2. Complete the setup wizard:
   - Organization information
   - Admin account
   - Server info (URL)
3. Set up your workspace

## Key Features

### Omnichannel (Customer Service)

1. Navigate to **Administration > Omnichannel**
2. Enable omnichannel
3. Add channels: Live Chat widget, Email, WhatsApp, SMS

### Live Chat Widget

```html
<!-- Add to your website -->
<script type="text/javascript">
(function(w, d, s, u) {
  w.RocketChat = function(c) { w.RocketChat._.push(c) };
  w.RocketChat._ = [];
  w.RocketChat.url = u;
  var h = d.getElementsByTagName(s)[0], j = d.createElement(s);
  j.async = true;
  j.src = u + '/livechat/rocketchat-livechat.min.js?_=201903270000';
  h.parentNode.insertBefore(j, h);
})(window, document, 'script', 'https://chat.example.com');
</script>
```

### Video Calling with Jitsi

```yaml
services:
  rocketchat:
    environment:
      OVERWRITE_SETTING_Jitsi_Enabled: "true"
      OVERWRITE_SETTING_Jitsi_Domain: meet.jit.si
      OVERWRITE_SETTING_Jitsi_URL_Room_Prefix: chat_
```

## Conclusion

Rocket.Chat deployed via Portainer provides a comprehensive team communication platform that goes beyond simple chat. Its omnichannel capabilities make it suitable for customer service use cases, while the open API and webhooks enable extensive integration with other tools. MongoDB replica set configuration is required for proper Rocket.Chat operation.
