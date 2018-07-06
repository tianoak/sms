# sms
receive new messages from android phone and read messages from inbox


//Main_Activity

package com.qa.sms;

import android.content.ComponentName;
import android.content.ContentResolver;
import android.content.Intent;
import android.database.Cursor;
import android.database.sqlite.SQLiteException;
import android.net.Uri;
import android.support.v4.app.ActivityCompat;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.widget.Button;
import android.widget.ImageView;
import android.widget.ScrollView;
import android.widget.TextView;
import android.widget.Toast;

import java.text.SimpleDateFormat;
import java.util.Date;

public class MainActivity extends AppCompatActivity {

    public final int REQUEST_CODE_ASK_PERMISSIONS = 1001;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ActivityCompat.requestPermissions(MainActivity.this, new String[]{"android.permission.READ_SMS"}, REQUEST_CODE_ASK_PERMISSIONS);
    }

    //这里是在登录界面label上右上角添加三个点，里面可添加其他功能
    @Override
    public boolean onCreateOptionsMenu(Menu menu)
    {
        // Inflate the menu; this adds items to the action bar if it is present.
        getMenuInflater().inflate(R.menu.main, menu);//这里是调用menu文件夹中的main.xml，在登陆界面label右上角的三角里显示其他功能
        return true;
    }

    public boolean onOptionsItemSelected(MenuItem item) {
        final TextView tv = new TextView(this);
        final ScrollView sv = new ScrollView(this);
        tv.setText(getSmsInPhone());
        sv.addView(tv);
        setContentView(sv);
        return true;
    }

    public String getSmsInPhone() {

        final String SMS_URI_ALL = "content://sms/";
        final String SMS_URI_INBOX = "content://sms/inbox";
        final String SMS_URI_SEND = "content://sms/sent";
        final String SMS_URI_DRAFT = "content://sms/draft";
        final String SMS_URI_CONV = "content://sms/conversations";

        StringBuilder smsBuilder = new StringBuilder();

        try{
            ContentResolver cr = getContentResolver();
            String[] projection = new String[]{"_id", "address", "person",
                    "body", "date", "type"};
            Uri uri = Uri.parse(SMS_URI_ALL);
            Cursor cur = cr.query(uri, projection, null, null, "date desc");

            if (cur.moveToFirst()) {

                int index_Address = cur.getColumnIndex("address");
                int index_Person = cur.getColumnIndex("person");
                int index_Body = cur.getColumnIndex("body");
                int index_Date = cur.getColumnIndex("date");
                int index_Type = cur.getColumnIndex("type");


                do {
                    String strAddress = cur.getString(index_Address);
                    int intPerson = cur.getInt(index_Person);
                    String strbody = cur.getString(index_Body);
                    long longDate = cur.getLong(index_Date);
                    int intType = cur.getInt(index_Type);

                    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                    Date d = new Date(longDate);
                    String strDate = dateFormat.format(d);

                    String strType = "";
                    if (intType == 1) {
                        strType = "接收";
                    } else if (intType == 2) {
                        strType = "发送";
                    } else {
                        strType = "null";
                    }

                    smsBuilder.append("[ ");
                    smsBuilder.append(strAddress + ", ");
                    smsBuilder.append(intPerson + ", ");
                    smsBuilder.append(strbody + ", ");
                    smsBuilder.append(strDate + ", ");
                    smsBuilder.append(strType);
                    smsBuilder.append(" ]\n\n");
                } while (cur.moveToNext());
            } else {
                smsBuilder.append("no result!\n\n");
            }

            smsBuilder.append("getSmsInPhone has executed!");
        } catch(SQLiteException ex) {
            Log.d("SQLiteException in getSmsInPhone", ex.getMessage());
        }
        return smsBuilder.toString();
    }
}



//SmsReceiver
package com.qa.sms;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.os.Bundle;
import android.os.Looper;
import android.telephony.SmsMessage;
import android.util.Log;
import android.widget.Toast;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.UnsupportedEncodingException;
import java.net.HttpURLConnection;
import java.net.MalformedURLException;
import java.net.ProtocolException;
import java.net.URL;
import java.net.URLEncoder;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.stream.Collectors;

public class SmsReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        Bundle intentExtras = intent.getExtras();
        if (intentExtras!= null) {

            /* Get Messages */
            Object[] sms = (Object[]) intentExtras.get("pdus");
            String format = intentExtras.getString("format");
            SmsMessage[] messages = new SmsMessage[sms.length];

            for (int i = 0; i < messages.length; i++) {
                //使用SmsMessage()的静态方法createFromPdu()方法将每一个pdu字节数组转换为SmsMessage对象
                messages[i] = SmsMessage.createFromPdu((byte[]) sms[i], format);
            }
            //调用SmsMessage对象的getOriginatingAddress()方法就可以获取到短信的发送发号码
            String address = messages[0].getOriginatingAddress();
            String fullMessage = "";

            for (SmsMessage message : messages) {
                //调用SmsMessage对象的getMessageBody()方法就可以获取到短信的内容，将每一个SmsMessage对象中的短信内容拼接起来，就组成了一条完整的短信内容
                fullMessage += message.getMessageBody();
            }

            //get phone and time
            final String phone = messages[0].getOriginatingAddress();
            final Long time = messages[0].getTimestampMillis();
            SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            Date d = new Date(time);
            final String strDate = dateFormat.format(d);

            //Log.d("receive messages time: ", strDate);
            //Log.d("receive new messages: ", fullMessage);

            final String finalFullMessage = fullMessage;

            new Thread(new Runnable(){
                @Override
                public void run() {
                    try {
                        Looper.prepare();
                        send_sms(strDate, finalFullMessage);
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                }
            }).start();


            Toast.makeText(context, phone + ":" + fullMessage + "\n resend" , Toast.LENGTH_SHORT).show();
        }
        //abortBroadcast();
    }


    public void send_sms(String date, String message) throws UnsupportedEncodingException {

        String content = URLEncoder.encode(message, "UTF-8");
        String path = "http:?content="+content+"&date="+date;
        //Log.d("url: ", path);
        try {
            URL url = new URL(path);
            HttpURLConnection connection = (HttpURLConnection) url.openConnection();
            // set connection output to true
            //connection.setDoOutput(true);
            connection.setConnectTimeout(5000);
            connection.setRequestMethod("GET");
            //获得结果码

            int responseCode = connection.getResponseCode();
            InputStream is = connection.getInputStream();

                if(responseCode == HttpURLConnection.HTTP_OK){
                    //请求成功 获得返回的流
                Log.d("resend status",  "OK");
                //return res+"/n/n resend OK";
                //Toast.makeText(con, result + "/n/n resend OK", Toast.LENGTH_SHORT).show();
                //Log.d("get request: ", result + "/n/n resend OK");
            }else {
                //请求失败
                Log.d("Error Code", String.valueOf(responseCode));
                //return res+"/n/n resend ERROR" +"/n Error Code: "+String.valueOf(responseCode);
                //Toast.makeText(con, result + "/n/n resend ERROR", Toast.LENGTH_SHORT).show();
                //Log.d("error code: ", String.valueOf(responseCode));
                // return null;
            }
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (ProtocolException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
       // return "getResponseCode ERROR";
    }
}

//Manifest
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.qa.sms">

    <uses-permission android:name="android.permission.READ_SMS"/>
    <uses-permission android:name="android.permission.RECEIVE_SMS"/>
    <uses-permission android:name="android.permission.INTERNET"/>

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <receiver android:name=".SmsReceiver" android:exported="true">
            <intent-filter android:priority="1000">
                <action android:name="android.provider.Telephony.SMS_RECEIVED"/>
            </intent-filter>
        </receiver>

    </application>

</manifest>

//res/layout/activity_main
<?xml version="1.0" encoding="utf-8"?>

<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

</android.support.constraint.ConstraintLayout>

//res/menu/main
<menu xmlns:android="http://schemas.android.com/apk/res/android" >

    <item
        android:id="@+id/action_settings"
        android:orderInCategory="100"
        android:title="refresh"/>

</menu>
