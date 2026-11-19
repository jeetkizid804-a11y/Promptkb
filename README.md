name: Build APK
on:
  push:
    branches: [main, master]
  workflow_dispatch:
permissions:
  contents: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: 8.9
      - name: Create project files
        run: |
          cat > settings.gradle.kts <<'EOF_PK'
          pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = "PromptKeyboard"
          include(":app")
          EOF_PK
          cat > build.gradle.kts <<'EOF_PK'
          plugins {
              id("com.android.application") version "8.5.2" apply false
              id("org.jetbrains.kotlin.android") version "1.9.24" apply false
          }
          EOF_PK
          cat > gradle.properties <<'EOF_PK'
          org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8
          kotlin.code.style=official
          EOF_PK
          mkdir -p app
          cat > app/build.gradle.kts <<'EOF_PK'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }

          android {
              namespace = "com.promptkb.app"
              compileSdk = 34

              defaultConfig {
                  applicationId = "com.promptkb.app"
                  minSdk = 24
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions {
                  jvmTarget = "17"
              }
          }
          EOF_PK
          mkdir -p app/src/main
          cat > app/src/main/AndroidManifest.xml <<'EOF_PK'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">

              <application
                  android:allowBackup="true"
                  android:label="Prompt Keyboard"
                  android:theme="@android:style/Theme.DeviceDefault.Light">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <service
                      android:name=".PromptKeyboardService"
                      android:exported="true"
                      android:label="Prompt Keyboard"
                      android:permission="android.permission.BIND_INPUT_METHOD">
                      <intent-filter>
                          <action android:name="android.view.InputMethod" />
                      </intent-filter>
                      <meta-data
                          android:name="android.view.im"
                          android:resource="@xml/method" />
                  </service>
              </application>
          </manifest>
          EOF_PK
          mkdir -p app/src/main/res/xml
          cat > app/src/main/res/xml/method.xml <<'EOF_PK'
          <?xml version="1.0" encoding="utf-8"?>
          <input-method xmlns:android="http://schemas.android.com/apk/res/android"
              android:settingsActivity="com.promptkb.app.MainActivity" />
          EOF_PK
          mkdir -p app/src/main/java/com/promptkb/app
          cat > app/src/main/java/com/promptkb/app/Store.kt <<'EOF_PK'
          package com.promptkb.app

          import android.content.Context
          import org.json.JSONArray
          import org.json.JSONObject

          data class Item(val n: Int, val t: String)

          object Store {
              private const val PREFS = "pk"

              fun parse(raw: String, strip: Boolean, single: Boolean): List<Item> {
                  val text = raw.replace("\r", "")
                  val re = Regex(
                      "^[ \\t]*Prompt[ \\t]*(\\d+)[ \\t]*[:\\uFF1A.\\-\\u2013][ \\t]*",
                      setOf(RegexOption.MULTILINE, RegexOption.IGNORE_CASE)
                  )
                  val out = ArrayList<Item>()

                  fun fin(n: Int, body: String) {
                      var s = body.trim()
                      if (s.isEmpty()) return
                      if (single) s = s.replace(Regex("\\s*\\n\\s*"), " ")
                      if (!strip) s = "Prompt $n: $s"
                      out.add(Item(n, s))
                  }

                  val ms = re.findAll(text).toList()
                  if (ms.isNotEmpty()) {
                      for (i in ms.indices) {
                          val m = ms[i]
                          val start = m.range.last + 1
                          val end = if (i + 1 < ms.size) ms[i + 1].range.first else text.length
                          fin(m.groupValues[1].toInt(), text.substring(start, end))
                      }
                  } else {
                      var parts = text.split(Regex("\\n\\s*\\n")).map { it.trim() }.filter { it.isNotEmpty() }
                      if (parts.size < 2) parts = text.split("\n").map { it.trim() }.filter { it.isNotEmpty() }
                      parts.forEachIndexed { i, p -> fin(i + 1, p) }
                  }
                  return out
              }

              fun save(ctx: Context, items: List<Item>) {
                  val arr = JSONArray()
                  for (it in items) arr.put(JSONObject().put("n", it.n).put("t", it.t))
                  ctx.getSharedPreferences(PREFS, Context.MODE_PRIVATE).edit().putString("items", arr.toString()).apply()
              }

              fun load(ctx: Context): MutableList<Item> {
                  val out = ArrayList<Item>()
                  try {
                      val s = ctx.getSharedPreferences(PREFS, Context.MODE_PRIVATE).getString("items", null) ?: return out
                      val arr = JSONArray(s)
                      for (i in 0 until arr.length()) {
                          val o = arr.getJSONObject(i)
                          out.add(Item(o.getInt("n"), o.getString("t")))
                      }
                  } catch (e: Exception) {
                  }
                  return out
              }

              fun getReplace(ctx: Context): Boolean =
                  ctx.getSharedPreferences(PREFS, Context.MODE_PRIVATE).getBoolean("replace", false)

              fun setReplace(ctx: Context, v: Boolean) {
                  ctx.getSharedPreferences(PREFS, Context.MODE_PRIVATE).edit().putBoolean("replace", v).apply()
              }
          }
          EOF_PK
          mkdir -p app/src/main/java/com/promptkb/app
          cat > app/src/main/java/com/promptkb/app/PromptKeyboardService.kt <<'EOF_PK'
          package com.promptkb.app

          import android.content.Context
          import android.graphics.Color
          import android.inputmethodservice.InputMethodService
          import android.os.Handler
          import android.os.Looper
          import android.text.TextUtils
          import android.view.KeyEvent
          import android.view.View
          import android.view.ViewGroup
          import android.view.inputmethod.EditorInfo
          import android.view.inputmethod.InputMethodManager
          import android.widget.BaseAdapter
          import android.widget.Button
          import android.widget.LinearLayout
          import android.widget.ListView
          import android.widget.TextView
          import android.widget.Toast

          class PromptKeyboardService : InputMethodService() {

              private var items: MutableList<Item> = ArrayList()
              private lateinit var adapter: ItemAdapter
              private lateinit var list: ListView
              private lateinit var info: TextView
              private lateinit var delAll: Button
              private lateinit var replaceBtn: Button
              private val handler = Handler(Looper.getMainLooper())
              private var armed = false
              private var replaceMode = false

              private fun dp(v: Int) = (v * resources.displayMetrics.density).toInt()

              override fun onEvaluateFullscreenMode(): Boolean = false

              override fun onCreateInputView(): View {
                  replaceMode = Store.getReplace(this)
                  items = Store.load(this)

                  val root = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setBackgroundColor(0xFF1E1E1E.toInt())
                      layoutParams = ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, dp(320))
                  }

                  val bar = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL }
                  fun key(label: String, onClick: () -> Unit): Button {
                      val b = Button(this).apply {
                          text = label
                          textSize = 12f
                          isAllCaps = false
                          setPadding(0, 0, 0, 0)
                          setOnClickListener { onClick() }
                      }
                      bar.addView(b, LinearLayout.LayoutParams(0, dp(44), 1f))
                      return b
                  }

                  val globe = key("ABC") { switchKeyboard() }
                  globe.setOnLongClickListener {
                      (getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager).showInputMethodPicker()
                      true
                  }
                  key("⌫") { sendDownUpKeyEvents(KeyEvent.KEYCODE_DEL) }
                  key("Clear") { clearField() }
                  key("Enter") { sendDownUpKeyEvents(KeyEvent.KEYCODE_ENTER) }
                  replaceBtn = key("") { toggleReplace() }
                  delAll = key("Delete all") { onDeleteAll() }
                  updateReplaceLabel()
                  root.addView(bar, LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT))

                  info = TextView(this).apply {
                      setTextColor(0xFFB0B0B0.toInt())
                      textSize = 12f
                      setPadding(dp(12), dp(4), dp(12), dp(4))
                  }
                  root.addView(info)

                  adapter = ItemAdapter()
                  list = ListView(this).apply {
                      this.adapter = this@PromptKeyboardService.adapter
                      dividerHeight = 1
                      setOnItemClickListener { _, _, pos, _ -> use(pos) }
                      setOnItemLongClickListener { _, _, pos, _ -> removeAt(pos); true }
                  }
                  root.addView(list, LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 0, 1f))

                  updateInfo()
                  return root
              }

              override fun onStartInputView(editorInfo: EditorInfo?, restarting: Boolean) {
                  super.onStartInputView(editorInfo, restarting)
                  if (::adapter.isInitialized) {
                      items = Store.load(this)
                      adapter.notifyDataSetChanged()
                      list.setSelection(0)
                      updateInfo()
                  }
              }

              private fun switchKeyboard() {
                  val imm = getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
                  val ok = try { switchToPreviousInputMethod() } catch (e: Exception) { false }
                  if (!ok) imm.showInputMethodPicker()
              }

              private fun clearField() {
                  val ic = currentInputConnection ?: return
                  ic.performContextMenuAction(android.R.id.selectAll)
                  ic.commitText("", 1)
                  ic.deleteSurroundingText(100000, 100000)
              }

              private fun use(pos: Int) {
                  if (pos < 0 || pos >= items.size) return
                  val ic = currentInputConnection ?: return
                  val item = items[pos]
                  if (replaceMode) clearField()
                  ic.commitText(item.t, 1)
                  items.removeAt(pos)
                  items.add(item)
                  Store.save(this, items)
                  adapter.notifyDataSetChanged()
                  list.setSelection(0)
                  updateInfo()
              }

              private fun removeAt(pos: Int) {
                  if (pos < 0 || pos >= items.size) return
                  val item = items.removeAt(pos)
                  Store.save(this, items)
                  adapter.notifyDataSetChanged()
                  updateInfo()
                  Toast.makeText(this, "#" + item.n + " hata diya", Toast.LENGTH_SHORT).show()
              }

              private fun onDeleteAll() {
                  if (!armed) {
                      armed = true
                      delAll.text = "Pakka? Dobara dabayein"
                      handler.postDelayed({ armed = false; delAll.text = "Delete all" }, 3000)
                  } else {
                      armed = false
                      handler.removeCallbacksAndMessages(null)
                      delAll.text = "Delete all"
                      items.clear()
                      Store.save(this, items)
                      adapter.notifyDataSetChanged()
                      updateInfo()
                  }
              }

              private fun toggleReplace() {
                  replaceMode = !replaceMode
                  Store.setReplace(this, replaceMode)
                  updateReplaceLabel()
              }

              private fun updateReplaceLabel() {
                  replaceBtn.text = if (replaceMode) "Replace: ON" else "Replace: OFF"
              }

              private fun updateInfo() {
                  info.text = if (items.isEmpty()) "Koi prompt nahi. Pehle app kholkar import karein."
                  else "Total " + items.size + "  |  Agla: #" + items[0].n + "  (tap = paste, long-press = hatao)"
              }

              private inner class ItemAdapter : BaseAdapter() {
                  override fun getCount() = items.size
                  override fun getItem(position: Int) = items[position]
                  override fun getItemId(position: Int) = position.toLong()
                  override fun getView(position: Int, convertView: View?, parent: ViewGroup?): View {
                      val tv = (convertView as? TextView) ?: TextView(this@PromptKeyboardService).apply {
                          setTextColor(Color.WHITE)
                          textSize = 14f
                          maxLines = 2
                          ellipsize = TextUtils.TruncateAt.END
                          setPadding(dp(12), dp(10), dp(12), dp(10))
                      }
                      val item = items[position]
                      tv.text = "#" + item.n + "  " + item.t
                      tv.setBackgroundColor(
                          if (position == 0) 0xFF1F4E8C.toInt()
                          else if (position % 2 == 0) 0xFF262626.toInt() else 0xFF2E2E2E.toInt()
                      )
                      return tv
                  }
              }
          }
          EOF_PK
          mkdir -p app/src/main/java/com/promptkb/app
          cat > app/src/main/java/com/promptkb/app/MainActivity.kt <<'EOF_PK'
          package com.promptkb.app

          import android.app.Activity
          import android.content.Context
          import android.content.Intent
          import android.os.Bundle
          import android.provider.Settings
          import android.view.Gravity
          import android.view.View
          import android.view.inputmethod.InputMethodManager
          import android.widget.Button
          import android.widget.CheckBox
          import android.widget.EditText
          import android.widget.LinearLayout
          import android.widget.ScrollView
          import android.widget.TextView
          import android.widget.Toast

          class MainActivity : Activity() {
              private lateinit var status: TextView
              private lateinit var cbStrip: CheckBox
              private lateinit var cbSingle: CheckBox
              private lateinit var paste: EditText

              private fun dp(v: Int) = (v * resources.displayMetrics.density).toInt()

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  val col = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setPadding(dp(16), dp(16), dp(16), dp(16))
                  }
                  fun add(v: View) {
                      col.addView(v, LinearLayout.LayoutParams(-1, -2).apply { topMargin = dp(8) })
                  }
                  fun btn(t: String, a: () -> Unit): Button {
                      val b = Button(this)
                      b.text = t
                      b.isAllCaps = false
                      b.setOnClickListener { a() }
                      return b
                  }

                  val title = TextView(this)
                  title.text = "Prompt Keyboard"
                  title.textSize = 22f
                  add(title)

                  status = TextView(this)
                  add(status)

                  add(btn("1. Keyboard enable karein") {
                      startActivity(Intent(Settings.ACTION_INPUT_METHOD_SETTINGS))
                  })
                  add(btn("2. Is keyboard ko chunein") {
                      (getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager).showInputMethodPicker()
                  })

                  cbStrip = CheckBox(this)
                  cbStrip.text = "\"Prompt N:\" label hata dein"
                  cbStrip.isChecked = true
                  add(cbStrip)

                  cbSingle = CheckBox(this)
                  cbSingle.text = "Multi-line prompt ko ek line banayein"
                  cbSingle.isChecked = true
                  add(cbSingle)

                  add(btn("3. TXT file import karein") {
                      val i = Intent(Intent.ACTION_OPEN_DOCUMENT)
                      i.addCategory(Intent.CATEGORY_OPENABLE)
                      i.type = "*/*"
                      startActivityForResult(i, 1)
                  })

                  paste = EditText(this)
                  paste.hint = "Ya yahan prompts paste karein"
                  paste.minLines = 4
                  paste.gravity = Gravity.TOP
                  add(paste)
                  add(btn("Paste kiya hua text import karein") { doImport(paste.text.toString()) })

                  add(btn("Keyboard ke saare prompts hata dein") {
                      Store.save(this, emptyList())
                      refresh()
                  })

                  val sv = ScrollView(this)
                  sv.addView(col)
                  setContentView(sv)
                  refresh()
              }

              override fun onResume() {
                  super.onResume()
                  refresh()
              }

              @Deprecated("Deprecated in Java")
              override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
                  super.onActivityResult(requestCode, resultCode, data)
                  if (requestCode == 1 && resultCode == RESULT_OK) {
                      val uri = data?.data ?: return
                      try {
                          val t = contentResolver.openInputStream(uri)!!.bufferedReader().use { it.readText() }
                          doImport(t)
                      } catch (e: Exception) {
                          Toast.makeText(this, "File nahi parhi ja saki", Toast.LENGTH_LONG).show()
                      }
                  }
              }

              private fun doImport(text: String) {
                  val list = Store.parse(text, cbStrip.isChecked, cbSingle.isChecked)
                  if (list.isEmpty()) {
                      Toast.makeText(this, "Koi prompt nahi mila", Toast.LENGTH_LONG).show()
                      return
                  }
                  Store.save(this, list)
                  refresh()
                  Toast.makeText(this, list.size.toString() + " prompts import hue", Toast.LENGTH_LONG).show()
              }

              private fun refresh() {
                  status.text = Store.load(this).size.toString() + " prompts keyboard mein hain"
              }
          }
          EOF_PK
      - name: Build
        run: gradle assembleDebug --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: PromptKeyboard-apk
          path: app/build/outputs/apk/debug/app-debug.apk
      - uses: softprops/action-gh-release@v2
        with:
          tag_name: apk-${{ github.run_number }}
          name: Prompt Keyboard APK ${{ github.run_number }}
          files: app/build/outputs/apk/debug/app-debug.apk
