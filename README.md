# Chat Module with WebSocket Using Ratchet (http://socketo.me/)

This repository contains a chat module implemented using the Ratchet WebSocket package for real-time one-to-one communication.

## Steps

### 1. Update Nginx Configuration

Update your Nginx configuration to support WebSocket connections. Edit your site configuration file (e.g., `/etc/nginx/sites-available/default` or your virtual host configuration) and add the following within the `server` block:

```nginx
location ~ /WsConn {
    proxy_pass             http://127.0.0.1:8083;
    proxy_read_timeout     60;
    proxy_connect_timeout  60;
    proxy_redirect         off;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
}
```

### 2. Restart Nginx

Apply the changes by restarting Nginx:

```bash
sudo systemctl restart nginx
```

### 3. Create Socket Server Script

Create a script to run the socket server:

```php
public function index()
{
    if (!is_cli()) {
        die('Don\'t be sneaky.');
    }
    CLI::write('Starting chat server...', 'green');
    $server = IoServer::factory(
        new HttpServer(
            new WsServer(
                new Chat()
            )
        ),
        8083
    );

    $server->run();
}
```

### 4. Server Connection Library Code

Implement the server-side logic:

```php
<?php

namespace App\Libraries;

use Ratchet\MessageComponentInterface;
use Ratchet\ConnectionInterface;
use CodeIgniter\Database\Config;

class Chat implements MessageComponentInterface
{
    protected $clients;
    protected $users;
    protected $userResources;
    protected $db;

    public function __construct()
    {
        $this->clients = new \SplObjectStorage;
        $this->users = [];
        $this->userResources = [];
        $this->db = Config::connect();
    }

    public function onOpen(ConnectionInterface $conn)
    {
        $this->clients->attach($conn);
        echo "New connection! ({$conn->resourceId})\n";
    }

    public function onMessage(ConnectionInterface $from, $msg)
    {
        $data = json_decode($msg);

        if (isset($data->command)) {
            switch ($data->command) {
                case "register":
                    $this->registerUser($from, $data->userId);
                    break;
                case "message":
                    $this->sendMessage($from, $data);
                    break;
                default:
                    break;
            }
        }
    }

    public function onClose(ConnectionInterface $conn)
    {
        $this->clients->detach($conn);
        foreach ($this->userResources as &$resources) {
            $key = array_search($conn->resourceId, $resources);
            if ($key !== false) {
                unset($resources[$key]);
            }
        }
        echo "Connection {$conn->resourceId} has disconnected\n";
    }

    public function onError(ConnectionInterface $conn, \Exception $e)
    {
        echo "An error has occurred: {$e->getMessage()}\n";
        $conn->close();
    }

    protected function registerUser(ConnectionInterface $conn, $userId)
    {
        if (!isset($this->userResources[$userId])) {
            $this->userResources[$userId] = [];
        }
        $this->userResources[$userId][] = $conn->resourceId;
        $this->users[$conn->resourceId] = $conn;
        echo "User $userId registered with resourceId {$conn->resourceId}\n";
    }

    protected function sendMessage(ConnectionInterface $from, $data)
    {
        if (isset($this->userResources[$data->to])) {
            foreach ($this->userResources[$data->to] as $resourceId) {
                if (isset($this->users[$resourceId])) {
                    $this->users[$resourceId]->send(json_encode($data));
                }
            }
        }
    }

    protected function getUsernameById($userId)
    {
        $query = $this->db->table('users')
            ->select('username')
            ->where('id', $userId)
            ->get();

        $result = $query->getRow();
        return $result ? $result->username : null;
    }
}
```

### 5. Define Routes

Add the following routes:

```php
$routes->get('/chats', 'Chat::index', ['filter' => 'auth', 'as' => 'chats']);
$routes->post('/chat/saveMessage', 'Chat::saveMessage', ['filter' => 'auth', 'as' => 'chats']);
$routes->get('chat/fetchChatHistory/(:num)', 'Chat::fetchChatHistory/$1');
$routes->post('chat/getUserByEmail', 'Chat::getUserByEmail');
$routes->get('chat/getPreviousChatParticipants', 'Chat::getPreviousChatParticipants');
$routes->cli('/server/start', 'ServerForChat::index');
```

### 6. Controller Code

Implement the controller logic:

```php
<?php

namespace App\Controllers;

use App\Models\ChatModel;
use App\Models\Users;

class Chat extends BaseController
{
    public function index()
    {
        $data['title'] = 'My Chat';
        $data['heading'] = 'My Chat';

        if (session()->get('user')['role'] == 'candidate' || session()->get('user')['role'] == 'company') {
            $data['main_content'] = 'common/chat';
        }

        return view('dashboard/template', $data);
    }

    public function getPreviousChatParticipants()
    {
        $currentUserId = session()->get('user')['id'];
        $chatModel = new ChatModel();
        $participants = $chatModel->getChatParticipants($currentUserId);
        return $this->response->setJSON(['success' => true, 'participants' => $participants]);
    }

    public function getUserById($userId)
    {
        $userModel = new UserModel();
        $user = $userModel->find($userId);
        if ($user) {
            return $this->response->setJSON(['success' => true, 'user' => $user]);
        } else {
            return $this->response->setJSON(['success' => false, 'message' => 'User not found']);
        }
    }

    public function getUserByEmail()
    {
        $email = $this->request->getJsonVar('email');
        $userModel = new Users();

        $user = $userModel->where('email', $email)->where('role', 'candidate')->first();

        if ($user) {
            return $this->response->setJSON(['success' => true, 'user' => $user]);
        } else {
            return $this->response->setJSON(['success' => false, 'message' => 'User not found.']);
        }
    }

    public function fetchChatHistory($receiverId)
    {
        $senderId = session()->get('user')['id'];
        $chatModel = new ChatModel();

        $chatHistory = $chatModel->where('(sender_id = ' . $senderId . ' AND receiver_id = ' . $receiverId . ') OR (sender_id = ' . $receiverId . ' AND receiver_id = ' . $senderId . ')')
            ->orderBy('created_at', 'ASC')
            ->findAll();

        return $this->response->setJSON($chatHistory);
    }

    public function saveMessage()
    {
        $id = session()->get('user')['id'];
        $chatModel = new ChatModel();
        $input = $this->request->getJSON();

        $data = [
            'sender_id' => $id,
            'receiver_id' => $input->to,
            'message' => $input->message,
            'created_at' => date('Y-m-d H:i:s')
        ];
        $chatModel->saveMessage($data);

        return $this->response->setJSON(['status' => 'success']);
    }
}
```

## Starting the WebSocket Server

Start the WebSocket server using the following command from the project/public folder:

```bash
php index.php server start
```

## Frontend Integration

Include the following HTML and CSS in your view file to create the chat interface:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Chat Application</title>
    <style>
        /* Add your CSS styles here */
    </style>
</head>
<body>
    <div class="row pb40">
        <div class="col-lg-6">
            <div class="dashboard_title_area">
                <h2><?= $heading ?></h2>
            </div>
        </div>
        <?php if (isset($user)) { ?>
            <div class="col-lg-6">
                <div class="text-lg-end">
                    <a href="<?= base_url('candidate/profile/' . $user['url_key']); ?>" target="_blank" class="ud-btn btn-dark default-box-shadow2">
                        View your profile on the website
                        <i class="fal fa-arrow-right-long"></i>
                    </a>
                </div>
            </div>
        <?php } ?>


    </div>

    <div id="chatParticipants">
        <h4>Recent Chats</h4>
        <div id="participants"></div>
        <div id="searchBox">
            <input type="text" id="searchInput" placeholder="Search user by email">
            <button onclick="searchUser()">Search</button>
        </div>
        <div id="searchResults"></div>
    </div>

    <div id="chatContainer">
        <div id="chatWindow"></div>
        <input type="text" id="messageInput" placeholder="Type your message...">
        <button onclick="sendMessage()">Send</button>
    </div>

    <script>
        var ws;
        var currentUser = <?= session()->get('user')['id'] ?>;
        var currentChatUser;

        window.onload = function () {
            ws = new WebSocket('ws://your_domain.com/WsConn');
            ws.onopen = function () {
                ws.send(JSON.stringify({ command: "register", userId: currentUser }));
            };

            ws.onmessage = function (event) {
                var data = JSON.parse(event.data);
                if (data.command === "message") {
                    displayMessage(data);
                }
            };

            loadParticipants();
        };

        function loadParticipants() {
            fetch('/chat/getPreviousChatParticipants')
                .then(response => response.json())
                .then(data => {
                    if (data.success) {
                        displayParticipants(data.participants);
                    }
                });
        }

        function displayParticipants(participants) {
            var participantsDiv = document.getElementById('participants');
            participantsDiv.innerHTML = '';
            participants.forEach(participant => {
                var participantDiv = document.createElement('div');
                participantDiv.innerText = participant.name;
                participantDiv.onclick = function () {
                    loadChatHistory(participant.id);
                };
                participantsDiv.appendChild(participantDiv);
            });
        }

        function loadChatHistory(userId) {
            currentChatUser = userId;
            fetch('/chat/fetchChatHistory/' + userId)
                .then(response => response.json())
                .then(messages => {
                    var chatWindow = document.getElementById('chatWindow');
                    chatWindow.innerHTML = '';
                    messages.forEach(message => {
                        displayMessage(message);
                    });
                });
        }

        function displayMessage(message) {
            var chatWindow = document.getElementById('chatWindow');
            var messageDiv = document.createElement('div');
            messageDiv.innerText = message.message;
            chatWindow.appendChild(messageDiv);
        }

        function sendMessage() {
            var messageInput = document.getElementById('messageInput');
            var message = messageInput.value;
            messageInput.value = '';

            var data = {
                command: "message",
                from: currentUser,
                to: currentChatUser,
                message: message
            };

            ws.send(JSON.stringify(data));

            fetch('/chat/saveMessage', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(data)
            });
        }

        function searchUser() {
            var email = document.getElementById('searchInput').value;
            fetch('/chat/getUserByEmail', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ email: email })
            })
                .then(response => response.json())
                .then(data => {
                    var searchResults = document.getElementById('searchResults');
                    searchResults.innerHTML = '';
                    if (data.success) {
                        var user = data.user;
                        var userDiv = document.createElement('div');
                        userDiv.innerText = user.name;
                        userDiv.onclick = function () {
                            loadChatHistory(user.id);
                        };
                        searchResults.appendChild(userDiv);
                    } else {
                        searchResults.innerText = 'User not found';
                    }
                });
        }
    </script>
</body>
</html>
```

This code provides a complete chat application using Ratchet for WebSocket communication, along with a search functionality to find users by email and retrieve chat histories.
