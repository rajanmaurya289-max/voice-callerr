name: Build Voice AI Assistant APK

on:
  push:
    branches: [main]
    paths: [".github/workflows/build-apk.yml"]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-24.04
    timeout-minutes: 25

    steps:
      - name: Set up Java
        uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: "17"

      - name: Use Android SDK
        shell: bash
        run: |
          voice_sdk_root="${ANDROID_HOME:-/usr/local/lib/android/sdk}"
          test -f "$voice_sdk_root/platforms/android-35/android.jar"
          test -d "$voice_sdk_root/build-tools/35.0.0"
          echo "ANDROID_HOME=$voice_sdk_root" >> "$GITHUB_ENV"
          echo "ANDROID_SDK_ROOT=$voice_sdk_root" >> "$GITHUB_ENV"

      - name: Set up Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.11.1"
          cache-disabled: true

      - name: Create complete assistant app
        shell: bash
        run: |
          python3 - <<'PYCODE'
          from pathlib import Path

          files = {
          "settings.gradle": r"""
          pluginManagement {
              repositories { google(); mavenCentral(); gradlePluginPortal() }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories { google(); mavenCentral() }
          }
          rootProject.name = 'VoiceAI'
          include ':app'
          """,

          "build.gradle": r"""
          plugins {
              id 'com.android.application' version '8.9.2' apply false
          }
          """,

          "app/build.gradle": r"""
          plugins { id 'com.android.application' }
          android {
              namespace 'com.example.voicecaller'
              compileSdk 35
              defaultConfig {
                  applicationId 'com.example.voicecaller'
                  minSdk 26
                  targetSdk 35
                  versionCode 4
                  versionName '4.0'
              }
              compileOptions {
                  sourceCompatibility JavaVersion.VERSION_17
                  targetCompatibility JavaVersion.VERSION_17
              }
          }
          """,

          "app/src/main/AndroidManifest.xml": r"""
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.RECORD_AUDIO"/>
              <uses-permission android:name="android.permission.READ_CONTACTS"/>
              <uses-permission android:name="android.permission.CALL_PHONE"/>
              <uses-permission android:name="android.permission.INTERNET"/>

              <queries>
                  <intent>
                      <action android:name="android.speech.RecognitionService"/>
                  </intent>
              </queries>

              <application
                  android:label="Voice AI"
                  android:allowBackup="false"
                  android:theme="@android:style/Theme.Material.Light.NoActionBar">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:launchMode="singleTop">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN"/>
                          <category android:name="android.intent.category.LAUNCHER"/>
                      </intent-filter>
                  </activity>

                  <activity
                      android:name=".AssistantActivity"
                      android:exported="true"
                      android:launchMode="singleTop"
                      android:excludeFromRecents="true"
                      android:theme="@android:style/Theme.Material.Light.Dialog.NoActionBar">
                      <intent-filter>
                          <action android:name="android.intent.action.ASSIST"/>
                          <action android:name="android.intent.action.VOICE_COMMAND"/>
                          <category android:name="android.intent.category.DEFAULT"/>
                      </intent-filter>
                  </activity>

                  <service
                      android:name=".VoiceAIService"
                      android:exported="true"
                      android:permission="android.permission.BIND_VOICE_INTERACTION">
                      <intent-filter>
                          <action android:name="android.service.voice.VoiceInteractionService"/>
                      </intent-filter>
                      <meta-data
                          android:name="android.voice_interaction"
                          android:resource="@xml/voice_ai"/>
                  </service>

                  <service
                      android:name=".VoiceAISessionService"
                      android:exported="true"
                      android:permission="android.permission.BIND_VOICE_INTERACTION"/>

                  <service
                      android:name=".SpeechBridgeService"
                      android:exported="true"
                      android:permission="android.permission.RECORD_AUDIO">
                      <intent-filter>
                          <action android:name="android.speech.RecognitionService"/>
                          <category android:name="android.intent.category.DEFAULT"/>
                      </intent-filter>
                  </service>
              </application>
          </manifest>
          """,

          "app/src/main/res/xml/voice_ai.xml": r"""
          <voice-interaction-service
              xmlns:android="http://schemas.android.com/apk/res/android"
              android:sessionService="com.example.voicecaller.VoiceAISessionService"
              android:recognitionService="com.example.voicecaller.SpeechBridgeService"
              android:settingsActivity="com.example.voicecaller.MainActivity"
              android:supportsAssist="true"
              android:supportsLaunchVoiceAssistFromKeyguard="false"
              android:supportsLocalInteraction="false"/>
          """,

          "app/src/main/java/com/example/voicecaller/VoiceAIService.java": r"""
          package com.example.voicecaller;
          import android.service.voice.VoiceInteractionService;
          import android.service.voice.VoiceInteractionSession;

          public class VoiceAIService extends VoiceInteractionService {
              @Override public void onReady() {
                  super.onReady();
                  setDisabledShowContext(
                      VoiceInteractionSession.SHOW_WITH_ASSIST |
                      VoiceInteractionSession.SHOW_WITH_SCREENSHOT);
              }
          }
          """,

          "app/src/main/java/com/example/voicecaller/VoiceAISessionService.java": r"""
          package com.example.voicecaller;
          import android.os.Bundle;
          import android.service.voice.VoiceInteractionSession;
          import android.service.voice.VoiceInteractionSessionService;

          public class VoiceAISessionService extends VoiceInteractionSessionService {
              @Override public VoiceInteractionSession onNewSession(Bundle args) {
                  return new VoiceAISession(this);
              }
          }
          """,

          "app/src/main/java/com/example/voicecaller/VoiceAISession.java": r"""
          package com.example.voicecaller;
          import android.content.Context;
          import android.content.Intent;
          import android.os.Bundle;
          import android.os.Handler;
          import android.os.Looper;
          import android.service.voice.VoiceInteractionSession;
          import android.view.View;
          import android.widget.Button;
          import android.widget.LinearLayout;
          import android.widget.TextView;

          public class VoiceAISession extends VoiceInteractionSession {
              private final Handler handler = new Handler(Looper.getMainLooper());
              private TextView message;

              public VoiceAISession(Context context) { super(context); }

              @Override public View onCreateContentView() {
                  LinearLayout layout = new LinearLayout(getContext());
                  layout.setOrientation(LinearLayout.VERTICAL);
                  layout.setPadding(24, 24, 24, 24);
                  message = new TextView(getContext());
                  message.setText("Voice AI khul raha hai...");
                  layout.addView(message);
                  Button retry = new Button(getContext());
                  retry.setText("Bolo");
                  layout.addView(retry);
                  retry.setOnClickListener(v -> openPanel());
                  Button close = new Button(getContext());
                  close.setText("Band karo");
                  layout.addView(close);
                  close.setOnClickListener(v -> hide());
                  return layout;
              }

              @Override public void onShow(Bundle args, int flags) {
                  super.onShow(args, flags);
                  handler.post(this::openPanel);
              }

              private void openPanel() {
                  try {
                      startAssistantActivity(new Intent(getContext(),
                          AssistantActivity.class).addFlags(
                          Intent.FLAG_ACTIVITY_CLEAR_TOP |
                          Intent.FLAG_ACTIVITY_SINGLE_TOP));
                      hide();
                  } catch (RuntimeException e) {
                      if (message != null)
                          message.setText("Voice AI app kholkar permissions Allow karo.");
                  }
              }

              @Override public void onHide() {
                  handler.removeCallbacksAndMessages(null);
                  super.onHide();
              }

              @Override public void onDestroy() {
                  handler.removeCallbacksAndMessages(null);
                  super.onDestroy();
              }
          }
          """,

          "app/src/main/java/com/example/voicecaller/AssistantActivity.java": r"""
          package com.example.voicecaller;
          import android.content.Intent;
          import android.os.Bundle;

          public class AssistantActivity extends MainActivity {
              @Override public void onCreate(Bundle saved) {
                  getIntent().putExtra("auto_listen", true);
                  super.onCreate(saved);
              }

              @Override protected void onNewIntent(Intent intent) {
                  intent.putExtra("auto_listen", true);
                  super.onNewIntent(intent);
              }
          }
          """,

          "app/src/main/java/com/example/voicecaller/SpeechProvider.java": r"""
          package com.example.voicecaller;
          import android.content.ComponentName;
          import android.content.Context;
          import android.content.Intent;
          import android.content.pm.ApplicationInfo;
          import android.content.pm.ResolveInfo;
          import android.content.pm.ServiceInfo;
          import android.speech.RecognitionService;

          public final class SpeechProvider {
              private SpeechProvider() {}

              public static ComponentName find(Context context) {
                  ComponentName system = null, other = null;
                  for (ResolveInfo result : context.getPackageManager()
                      .queryIntentServices(new Intent(
                          RecognitionService.SERVICE_INTERFACE), 0)) {
                      ServiceInfo service = result.serviceInfo;
                      if (service == null || !service.enabled ||
                          !service.exported ||
                          service.packageName.equals(context.getPackageName()))
                          continue;
                      ComponentName component = new ComponentName(
                          service.packageName, service.name);
                      if (service.packageName.equals(
                          "com.google.android.googlequicksearchbox"))
                          return component;
                      if (system == null && (service.applicationInfo.flags &
                          ApplicationInfo.FLAG_SYSTEM) != 0)
                          system = component;
                      if (other == null) other = component;
                  }
                  return system != null ? system : other;
              }
          }
          """,

          "app/src/main/java/com/example/voicecaller/SpeechBridgeService.java": r"""
          package com.example.voicecaller;
          import android.content.ComponentName;
          import android.content.Intent;
          import android.os.Bundle;
          import android.os.RemoteException;
          import android.speech.RecognitionListener;
          import android.speech.RecognitionService;
          import android.speech.SpeechRecognizer;

          public class SpeechBridgeService extends RecognitionService {
              private SpeechRecognizer engine;
              private Callback current;
              private interface Reply {
                  void send(Callback callback) throws RemoteException;
              }

              private void reply(Callback expected, Reply action) {
                  if (current != expected) return;
                  try { action.send(expected); }
                  catch (RemoteException e) { release(); }
              }

              private void release() {
                  current = null;
                  if (engine != null) {
                      engine.destroy();
                      engine = null;
                  }
              }

              @Override protected void onStartListening(
                  Intent intent, Callback callback) {
                  release();
                  current = callback;
                  ComponentName provider = SpeechProvider.find(this);
                  if (provider == null) {
                      reply(callback, c -> c.error(SpeechRecognizer.ERROR_CLIENT));
                      release();
                      return;
                  }
                  engine = SpeechRecognizer.createSpeechRecognizer(this, provider);
                  engine.setRecognitionListener(new RecognitionListener() {
                      public void onReadyForSpeech(Bundle b) {
                          reply(callback, c -> c.readyForSpeech(b));
                      }
                      public void onBeginningOfSpeech() {
                          reply(callback, c -> c.beginningOfSpeech());
                      }
                      public void onRmsChanged(float r) {
                          reply(callback, c -> c.rmsChanged(r));
                      }
                      public void onBufferReceived(byte[] b) {
                          reply(callback, c -> c.bufferReceived(b));
                      }
                      public void onEndOfSpeech() {
                          reply(callback, c -> c.endOfSpeech());
                      }
                      public void onError(int e) {
                          reply(callback, c -> c.error(e));
                          if (current == callback) release();
                      }
                      public void onResults(Bundle b) {
                          reply(callback, c -> c.results(b));
                          if (current == callback) release();
                      }
                      public void onPartialResults(Bundle b) {
                          reply(callback, c -> c.partialResults(b));
                      }
                      public void onEvent(int type, Bundle b) {}
                  });
                  try { engine.startListening(new Intent(intent)); }
                  catch (RuntimeException e) {
                      reply(callback, c -> c.error(SpeechRecognizer.ERROR_CLIENT));
                      release();
                  }
              }

              @Override protected void onStopListening(Callback callback) {
                  if (current == callback && engine != null)
                      engine.stopListening();
              }

              @Override protected void onCancel(Callback callback) {
                  if (current == callback) release();
              }

              @Override public void onDestroy() {
                  release();
                  super.onDestroy();
              }
          }
          """,

          "app/src/main/java/com/example/voicecaller/MainActivity.java": r"""
          package com.example.voicecaller;

          import android.Manifest;
          import android.app.Activity;
          import android.app.AlertDialog;
          import android.app.role.RoleManager;
          import android.content.ComponentName;
          import android.content.Intent;
          import android.content.SharedPreferences;
          import android.content.pm.PackageManager;
          import android.database.Cursor;
          import android.net.Uri;
          import android.os.Build;
          import android.os.Bundle;
          import android.os.Handler;
          import android.os.Looper;
          import android.provider.ContactsContract.CommonDataKinds.Phone;
          import android.provider.Settings;
          import android.service.voice.VoiceInteractionService;
          import android.speech.RecognitionListener;
          import android.speech.RecognizerIntent;
          import android.speech.SpeechRecognizer;
          import android.view.View;
          import android.widget.ArrayAdapter;
          import android.widget.Button;
          import android.widget.CheckBox;
          import android.widget.EditText;
          import android.widget.LinearLayout;
          import android.widget.ScrollView;
          import android.widget.Spinner;
          import android.widget.TextView;
          import java.text.Normalizer;
          import java.util.ArrayList;
          import java.util.Arrays;
          import java.util.HashSet;
          import java.util.LinkedHashMap;
          import java.util.List;
          import java.util.Locale;
          import java.util.concurrent.ExecutorService;
          import java.util.concurrent.Executors;

          public class MainActivity extends Activity implements RecognitionListener {
              private final Handler handler = new Handler(Looper.getMainLooper());
              private final ExecutorService worker = Executors.newSingleThreadExecutor();
              private SpeechRecognizer recognizer;
              private TextView status, assistantState;
              private Button speak, search;
              private EditText typed;
              private Spinner language;
              private CheckBox direct;
              private SharedPreferences prefs;
              private boolean active, listening, autoListen;
              private int generation;
              private String pendingNumber;

              @Override public void onCreate(Bundle saved) {
                  super.onCreate(saved);
                  prefs = getSharedPreferences("voice_ai", MODE_PRIVATE);
                  autoListen = getIntent().getBooleanExtra("auto_listen", false);
                  boolean compact = this instanceof AssistantActivity;

                  ScrollView scroll = new ScrollView(this);
                  LinearLayout layout = new LinearLayout(this);
                  layout.setOrientation(LinearLayout.VERTICAL);
                  int pad = (int)(20 * getResources().getDisplayMetrics().density);
                  layout.setPadding(pad, pad, pad, pad);
                  if (!compact) {
                      scroll.setOnApplyWindowInsetsListener((v, insets) -> {
                          layout.setPadding(pad,
                              pad + insets.getSystemWindowInsetTop(),
                              pad, pad + insets.getSystemWindowInsetBottom());
                          return insets;
                      });
                  }
                  scroll.addView(layout);

                  TextView title = new TextView(this);
                  title.setText("Voice AI");
                  title.setTextSize(28);
                  layout.addView(title);

                  TextView help = new TextView(this);
                  help.setText(compact ?
                      "Saved contact ka naam bolo." :
                      "Permissions Allow karo, phir Voice AI ko default assistant banao.\n" +
                      "Power button ki Press & hold setting mein Digital assistant chuno.\n" +
                      "Default calling SIM ko SIM 1 rakho. Phone unlocked rakho.\n" +
                      "Is version mein custom wake word nahi hai.");
                  layout.addView(help);

                  assistantState = new TextView(this);
                  layout.addView(assistantState);

                  Button permissions = new Button(this);
                  permissions.setText("1. Permissions Allow karo");
                  layout.addView(permissions);
                  permissions.setOnClickListener(v -> requestPermissions(
                      new String[]{Manifest.permission.RECORD_AUDIO,
                          Manifest.permission.READ_CONTACTS,
                          Manifest.permission.CALL_PHONE}, 14));

                  Button choose = new Button(this);
                  choose.setText("2. Default assistant banao");
                  layout.addView(choose);
                  choose.setOnClickListener(v -> chooseAssistant());

                  Button settings = new Button(this);
                  settings.setText("3. Phone settings kholo");
                  layout.addView(settings);
                  settings.setOnClickListener(v -> {
                      try { startActivity(new Intent(Settings.ACTION_SETTINGS)); }
                      catch (RuntimeException e) {
                          status.setText("Phone Settings mein Power button search karo.");
                      }
                  });

                  language = new Spinner(this);
                  language.setAdapter(new ArrayAdapter<>(this,
                      android.R.layout.simple_spinner_dropdown_item,
                      new String[]{"English (India)", "Hindi (India)"}));
                  language.setSelection(prefs.getInt("language", 0));
                  layout.addView(language);

                  direct = new CheckBox(this);
                  direct.setText("Ek matching number ho to seedha call karo");
                  direct.setChecked(prefs.getBoolean("direct", true));
                  direct.setOnCheckedChangeListener((v, checked) ->
                      prefs.edit().putBoolean("direct", checked).apply());
                  layout.addView(direct);

                  speak = new Button(this);
                  speak.setText("Bolo");
                  layout.addView(speak);
                  speak.setOnClickListener(v -> {
                      if (listening) {
                          recognizer.cancel();
                          reset();
                          status.setText("Cancelled.");
                      } else beginVoice();
                  });

                  typed = new EditText(this);
                  typed.setSingleLine(true);
                  typed.setHint("Ya saved naam type karo");
                  layout.addView(typed);

                  search = new Button(this);
                  search.setText("Naam dhoondho");
                  layout.addView(search);
                  search.setOnClickListener(v -> {
                      if (!has(Manifest.permission.READ_CONTACTS)) {
                          requestPermissions(
                              new String[]{Manifest.permission.READ_CONTACTS}, 12);
                      } else lookup(typed.getText().toString());
                  });

                  status = new TextView(this);
                  status.setTextSize(18);
                  status.setText("Ready.");
                  layout.addView(status);
                  setContentView(scroll);

                  if (compact) {
                      for (View v : new View[]{assistantState, permissions,
                          choose, settings, language, direct})
                          v.setVisibility(View.GONE);
                  }
              }

              private boolean has(String permission) {
                  return checkSelfPermission(permission) ==
                      PackageManager.PERMISSION_GRANTED;
              }

              private void chooseAssistant() {
                  if (Build.VERSION.SDK_INT >= 29) {
                      RoleManager roles = getSystemService(RoleManager.class);
                      if (roles != null &&
                          roles.isRoleAvailable(RoleManager.ROLE_ASSISTANT)) {
                          if (roles.isRoleHeld(RoleManager.ROLE_ASSISTANT)) {
                              status.setText("Voice AI default assistant hai.");
                              return;
                          }
                          try {
                              startActivityForResult(roles.createRequestRoleIntent(
                                  RoleManager.ROLE_ASSISTANT), 15);
                              return;
                          } catch (RuntimeException ignored) {}
                      }
                  }
                  try {
                      startActivity(new Intent(Settings.ACTION_VOICE_INPUT_SETTINGS));
                  } catch (RuntimeException e) {
                      try {
                          startActivity(new Intent(
                              Settings.ACTION_MANAGE_DEFAULT_APPS_SETTINGS));
                      } catch (RuntimeException ignored) {
                          status.setText("Settings > Apps > Default apps > " +
                              "Digital assistant app mein Voice AI chuno.");
                      }
                  }
              }

              @Override protected void onNewIntent(Intent intent) {
                  super.onNewIntent(intent);
                  setIntent(intent);
                  if (intent.getBooleanExtra("auto_listen", false))
                      autoListen = true;
              }

              @Override protected void onStart() {
                  super.onStart();
                  active = true;
              }

              @Override protected void onResume() {
                  super.onResume();
                  boolean selected = VoiceInteractionService.isActiveService(this,
                      new ComponentName(this, VoiceAIService.class));
                  assistantState.setText(selected ?
                      "Default assistant: Voice AI" :
                      "Voice AI abhi default assistant nahi hai.");
                  if (autoListen) {
                      autoListen = false;
                      handler.postDelayed(() -> {
                          if (active && !listening) beginVoice();
                      }, 350);
                  }
              }

              private void beginVoice() {
                  prefs.edit().putInt("language",
                      language.getSelectedItemPosition()).apply();
                  if (!has(Manifest.permission.RECORD_AUDIO) ||
                      !has(Manifest.permission.READ_CONTACTS)) {
                      requestPermissions(new String[]{
                          Manifest.permission.RECORD_AUDIO,
                          Manifest.permission.READ_CONTACTS}, 10);
                      return;
                  }
                  ComponentName provider = SpeechProvider.find(this);
                  if (provider == null) {
                      status.setText("Speech service nahi mili. Google app " +
                          "enabled rakho, ya naam type karo.");
                      return;
                  }
                  if (recognizer == null) {
                      recognizer = SpeechRecognizer.createSpeechRecognizer(
                          this, provider);
                      recognizer.setRecognitionListener(this);
                  }
                  Intent intent = new Intent(
                      RecognizerIntent.ACTION_RECOGNIZE_SPEECH);
                  intent.putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL,
                      RecognizerIntent.LANGUAGE_MODEL_FREE_FORM);
                  intent.putExtra(RecognizerIntent.EXTRA_LANGUAGE,
                      language.getSelectedItemPosition() == 0 ? "en-IN" : "hi-IN");
                  intent.putExtra(RecognizerIntent.EXTRA_MAX_RESULTS, 1);
                  intent.putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, false);
                  listening = true;
                  speak.setText("Cancel");
                  search.setEnabled(false);
                  status.setText("Sun raha hoon...");
                  try { recognizer.startListening(intent); }
                  catch (RuntimeException e) {
                      reset();
                      status.setText("Dobara Bolo dabao.");
                  }
              }

              private static String normalize(String value) {
                  if (value == null) return "";
                  return Normalizer.normalize(value, Normalizer.Form.NFC)
                      .toLowerCase(Locale.ROOT)
                      .replaceAll("[^\\p{L}\\p{M}\\p{N}\\s]", " ")
                      .trim().replaceAll("\\s+", " ");
              }

              private static String extractName(String sentence) {
                  String name = normalize(sentence);
                  name = name.replaceFirst(
                      "^(?:please\\s+)?(?:call|phone|dial|कॉल|फोन)\\s+", "");
                  name = name.replaceFirst(
                      "\\s+(?:(?:ko|को)\\s+)?(?:call|phone|कॉल|फोन)" +
                      "(?:\\s+(?:karo|kar|करो|कर|kijiye|कीजिए|do|दो))?$", "");
                  name = name.replaceFirst("\\s+(?:ko|को)$", "").trim();
                  if (name.matches("call|phone|dial|कॉल|फोन")) return "";
                  return name;
              }

              private static boolean wordMatch(String contact, String query) {
                  if (query.isEmpty()) return false;
                  return new HashSet<>(Arrays.asList(normalize(contact).split(" ")))
                      .containsAll(Arrays.asList(query.split(" ")));
              }

              private static class Contact {
                  final String name, number;
                  Contact(String name, String number) {
                      this.name = name; this.number = number;
                  }
              }

              private void lookup(String sentence) {
                  final int request = ++generation;
                  final String name = extractName(sentence);
                  if (name.isEmpty()) {
                      status.setText("Contact ka naam bolo.");
                      return;
                  }
                  status.setText("Suna: " + sentence + "\nDhoondh raha hoon...");
                  speak.setEnabled(false);
                  search.setEnabled(false);
                  worker.execute(() -> {
                      List<Contact> exact = new ArrayList<>();
                      List<Contact> partial = new ArrayList<>();
                      try (Cursor cursor = getContentResolver().query(
                          Phone.CONTENT_URI,
                          new String[]{Phone.DISPLAY_NAME, Phone.NUMBER},
                          null, null, Phone.DISPLAY_NAME + " ASC")) {
                          if (cursor != null) while (cursor.moveToNext()) {
                              String display = cursor.getString(0);
                              String raw = cursor.getString(1);
                              if (display == null || raw == null) continue;
                              String number = raw.replaceAll("[^+0-9]", "");
                              if (!raw.matches("[+0-9\\s().-]+") ||
                                  !number.matches("\\+?[0-9]{3,15}")) continue;
                              Contact c = new Contact(display, number);
                              if (normalize(display).equals(name)) exact.add(c);
                              else if (wordMatch(display, name)) partial.add(c);
                          }
                          List<Contact> candidates = exact.isEmpty() ?
                              partial : exact;
                          LinkedHashMap<String, Contact> unique =
                              new LinkedHashMap<>();
                          for (Contact c : candidates)
                              unique.put(normalize(c.name) + "\n" + c.number, c);
                          List<Contact> matches =
                              new ArrayList<>(unique.values());
                          runOnUiThread(() -> {
                              if (active && !isDestroyed() &&
                                  request == generation) {
                                  reset();
                                  showMatches(matches);
                              }
                          });
                      } catch (RuntimeException e) {
                          runOnUiThread(() -> {
                              if (active && !isDestroyed() &&
                                  request == generation) {
                                  reset();
                                  status.setText("Contacts permission check karo.");
                              }
                          });
                      }
                  });
              }

              private void showMatches(List<Contact> matches) {
                  if (matches.isEmpty()) {
                      status.setText("Naam nahi mila. Saved naam bolo, " +
                          "language badlo ya naam type karo.");
                  } else if (matches.size() == 1) {
                      Contact c = matches.get(0);
                      if (direct.isChecked()) call(c.number);
                      else confirm(c);
                  } else {
                      String[] labels = new String[matches.size()];
                      for (int i = 0; i < labels.length; i++)
                          labels[i] = matches.get(i).name + " — " +
                              matches.get(i).number;
                      new AlertDialog.Builder(this)
                          .setTitle("Kaunsa contact / number?")
                          .setItems(labels, (d, which) -> confirm(matches.get(which)))
                          .setNegativeButton("Cancel", null).show();
                  }
              }

              private void confirm(Contact c) {
                  new AlertDialog.Builder(this)
                      .setTitle("Call karein?")
                      .setMessage(c.name + "\n" + c.number)
                      .setPositiveButton("Call", (d, w) -> call(c.number))
                      .setNegativeButton("Cancel", null).show();
              }

              private void call(String number) {
                  if (!active) return;
                  if (!has(Manifest.permission.CALL_PHONE)) {
                      pendingNumber = number;
                      requestPermissions(
                          new String[]{Manifest.permission.CALL_PHONE}, 11);
                      return;
                  }
                  pendingNumber = null;
                  try {
                      startActivity(new Intent(Intent.ACTION_CALL,
                          Uri.fromParts("tel", number, null)));
                      status.setText("Call request bhej di.");
                      if (this instanceof AssistantActivity) finish();
                  } catch (RuntimeException e) {
                      status.setText("Phone app aur Call permission check karo.");
                  }
              }

              @Override public void onRequestPermissionsResult(
                  int request, String[] permissions, int[] results) {
                  super.onRequestPermissionsResult(request, permissions, results);
                  if (request == 10) {
                      if (has(Manifest.permission.RECORD_AUDIO) &&
                          has(Manifest.permission.READ_CONTACTS)) beginVoice();
                      else status.setText("Mic aur Contacts permission Allow karo.");
                  } else if (request == 11) {
                      String number = pendingNumber;
                      pendingNumber = null;
                      if (number != null && has(Manifest.permission.CALL_PHONE))
                          call(number);
                      else status.setText("Call permission Allow karo.");
                  } else if (request == 12) {
                      if (has(Manifest.permission.READ_CONTACTS))
                          lookup(typed.getText().toString());
                  } else if (request == 14) {
                      status.setText(has(Manifest.permission.RECORD_AUDIO) &&
                          has(Manifest.permission.READ_CONTACTS) &&
                          has(Manifest.permission.CALL_PHONE) ?
                          "Permissions ready. Ab Default assistant banao dabao." :
                          "Mic, Contacts aur Phone permissions Allow karo.");
                  }
              }

              private void reset() {
                  listening = false;
                  speak.setEnabled(true);
                  speak.setText("Bolo");
                  search.setEnabled(true);
              }

              @Override protected void onStop() {
                  active = false;
                  generation++;
                  pendingNumber = null;
                  handler.removeCallbacksAndMessages(null);
                  if (recognizer != null) recognizer.cancel();
                  reset();
                  super.onStop();
              }

              @Override protected void onDestroy() {
                  if (recognizer != null) recognizer.destroy();
                  worker.shutdownNow();
                  super.onDestroy();
              }

              @Override public void onResults(Bundle results) {
                  if (!listening) return;
                  reset();
                  if (!active) return;
                  ArrayList<String> words = results.getStringArrayList(
                      SpeechRecognizer.RESULTS_RECOGNITION);
                  if (words == null || words.isEmpty())
                      status.setText("Naam samajh nahi aaya. Dobara bolo.");
                  else lookup(words.get(0));
              }

              @Override public void onError(int error) {
                  if (!listening) return;
                  reset();
                  if (active) status.setText("Voice ruk gayi (code " + error +
                      "). Internet check karo aur dobara Bolo dabao.");
                         }

            @Override public void onReadyForSpeech(Bundle params) {
              if (active && listening)
                status.setText("Ab contact ka naam bolo.");
            }

            @Override public void onBeginningOfSpeech() {}
            @Override public void onRmsChanged(float rmsdB) {}
            @Override public void onBufferReceived(byte[] buffer) {}
            @Override public void onEndOfSpeech() {}
            @Override public void onPartialResults(Bundle results) {}
            @Override public void onEvent(int eventType, Bundle params) {}
          }
          """
          }

          for name, content in files.items():
              path = Path("voice-caller") / name
              path.parent.mkdir(parents=True, exist_ok=True)
              path.write_text(content.strip() + "\n", encoding="utf-8")
          PYCODE

      - name: Build APK
        working-directory: voice-caller
        run: gradle --no-daemon assembleDebug

      - name: Downloadable APK
        uses: actions/upload-artifact@v4
        with:
          name: VoiceAI-Assistant-APK
          path: voice-caller/app/build/outputs/apk/debug/app-debug.apk
          if-no-files-found: error
          retention-days: 7
