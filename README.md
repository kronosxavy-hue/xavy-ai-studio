package com.xavy.studio

import android.Manifest
import android.app.Activity
import android.content.Intent
import android.content.pm.PackageManager
import android.graphics.Color
import android.media.AudioFormat
import android.media.AudioRecord
import android.media.MediaPlayer
import android.media.MediaRecorder
import android.net.Uri
import android.os.Bundle
import android.view.Gravity
import android.widget.Button
import android.widget.LinearLayout
import android.widget.TextView
import java.io.File
import java.io.RandomAccessFile
import kotlin.math.abs

class MainActivity : Activity() {

    private val sampleRate = 44100
    private var beatUri: Uri? = null
    private var beatPlayer: MediaPlayer? = null
    private var takePlayer: MediaPlayer? = null
    private var recorder: AudioRecord? = null
    @Volatile private var recording = false
    private var takeCount = 0
    private var lastTake: File? = null

    private lateinit var status: TextView
    private lateinit var meter: TextView
    private lateinit var recBtn: Button

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val root = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            setBackgroundColor(Color.parseColor("#0D0D12"))
            setPadding(40, 80, 40, 40)
            gravity = Gravity.CENTER_HORIZONTAL
        }
        root.addView(TextView(this).apply {
            text = "XAVY AI STUDIO"
            textSize = 28f
            setTextColor(Color.parseColor("#FFC93C"))
            gravity = Gravity.CENTER
        })
        status = TextView(this).apply {
            text = "Étape 1 : import beat, enregistrement voix, lecture"
            setTextColor(Color.LTGRAY); gravity = Gravity.CENTER; setPadding(0, 30, 0, 30)
        }
        meter = TextView(this).apply {
            text = "Niveau : —"; setTextColor(Color.WHITE); gravity = Gravity.CENTER
        }
        root.addView(status); root.addView(meter)

        fun btn(label: String, action: () -> Unit) = Button(this).apply {
            text = label; setOnClickListener { action() }
        }.also { root.addView(it) }

        btn("Importer un beat") { pickBeat() }
        recBtn = btn("● Enregistrer (avec le beat)") { toggleRecording() }
        btn("▶ Réécouter la dernière prise") { playTake() }
        btn("■ Tout arrêter") { stopAll() }
        setContentView(root)

        if (checkSelfPermission(Manifest.permission.RECORD_AUDIO) != PackageManager.PERMISSION_GRANTED) {
            requestPermissions(arrayOf(Manifest.permission.RECORD_AUDIO), 1)
        }
    }

    private fun pickBeat() {
        val i = Intent(Intent.ACTION_OPEN_DOCUMENT).apply {
            addCategory(Intent.CATEGORY_OPENABLE); type = "audio/*"
        }
        startActivityForResult(i, 10)
    }

    @Deprecated("Deprecated in Java")
    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        if (requestCode == 10 && resultCode == RESULT_OK) {
            beatUri = data?.data
            status.text = "Beat chargé ✔"
        }
    }

    private fun toggleRecording() {
        if (recording) { stopRecording(); return }
        if (checkSelfPermission(Manifest.permission.RECORD_AUDIO) != PackageManager.PERMISSION_GRANTED) {
            status.text = "Autorise le micro dans les paramètres de l'app"; return
        }
        startRecording()
    }

    private fun startRecording() {
        val minBuf = AudioRecord.getMinBufferSize(
            sampleRate, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT)
        val rec = AudioRecord(MediaRecorder.AudioSource.MIC, sampleRate,
            AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT, minBuf * 2)
        if (rec.state != AudioRecord.STATE_INITIALIZED) { status.text = "Micro indisponible"; return }

        takeCount++
        val dir = getExternalFilesDir("takes") ?: filesDir
        dir.mkdirs()
        val file = File(dir, "prise_$takeCount.wav")
        lastTake = file
        recorder = rec
        recording = true
        recBtn.text = "■ Arrêter l'enregistrement"
        status.text = "Enregistrement…"

        beatUri?.let {
            beatPlayer?.release()
            beatPlayer = MediaPlayer().apply {
                setDataSource(this@MainActivity, it); prepare(); start()
            }
        }

        Thread {
            val raf = RandomAccessFile(file, "rw")
            raf.setLength(0)
            raf.write(ByteArray(44))
            val buf = ShortArray(minBuf)
            val bytes = ByteArray(minBuf * 2)
            var total = 0L
            rec.startRecording()
            while (recording) {
                val n = rec.read(buf, 0, buf.size)
                if (n <= 0) continue
                var peak = 0
                for (k in 0 until n) {
                    val s = buf[k].toInt()
                    bytes[2 * k] = (s and 0xFF).toByte()
                    bytes[2 * k + 1] = ((s shr 8) and 0xFF).toByte()
                    if (abs(s) > peak) peak = abs(s)
                }
                raf.write(bytes, 0, n * 2)
                total += n * 2
                val pct = peak * 100 / 32767
                runOnUiThread {
                    meter.text = if (pct >= 99) "Niveau : $pct % — SATURATION, baisse le gain"
                                 else "Niveau : $pct %"
                }
            }
            rec.stop(); rec.release()
            writeWavHeader(raf, total)
            raf.close()
            runOnUiThread { status.text = "Prise enregistrée : ${file.name}" }
        }.start()
    }

    private fun stopRecording() {
        recording = false
        beatPlayer?.release(); beatPlayer = null
        recBtn.text = "● Enregistrer (avec le beat)"
    }

    private fun writeWavHeader(raf: RandomAccessFile, dataLen: Long) {
        val byteRate = sampleRate * 2
        fun i32(v: Long) = byteArrayOf(v.toByte(), (v shr 8).toByte(), (v shr 16).toByte(), (v shr 24).toByte())
        fun i16(v: Int) = byteArrayOf(v.toByte(), (v shr 8).toByte())
        raf.seek(0)
        raf.write("RIFF".toByteArray()); raf.write(i32(36 + dataLen))
        raf.write("WAVE".toByteArray()); raf.write("fmt ".toByteArray())
        raf.write(i32(16)); raf.write(i16(1)); raf.write(i16(1))
        raf.write(i32(sampleRate.toLong())); raf.write(i32(byteRate.toLong()))
        raf.write(i16(2)); raf.write(i16(16))
        raf.write("data".toByteArray()); raf.write(i32(dataLen))
    }

    private fun playTake() {
        val f = lastTake
        if (f == null || !f.exists()) { status.text = "Aucune prise"; return }
        stopAll()
        takePlayer = MediaPlayer().apply { setDataSource(f.absolutePath); prepare(); start() }
        beatUri?.let {
            beatPlayer = MediaPlayer().apply { setDataSource(this@MainActivity, it); prepare(); start() }
        }
        status.text = "Lecture de ${f.name}"
    }

    private fun stopAll() {
        if (recording) stopRecording()
        takePlayer?.release(); takePlayer = null
        beatPlayer?.release(); beatPlayer = null
    }

    override fun onDestroy() {
        recording = false
        takePlayer?.release(); beatPlayer?.release()
        super.onDestroy()
    }
}<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <application
        android:label="XAVY AI STUDIO"
        android:allowBackup="true"
        android:theme="@android:style/Theme.DeviceDefault.NoActionBar">
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}
android {
    namespace = "com.xavy.studio"
    compileSdk = 34
    defaultConfig {
        applicationId = "com.xavy.studio"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "0.1.0"
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
}org.gradle.jvmargs=-Xmx2g
android.useAndroidX=false
kotlin.code.style=officialplugins {
    id("com.android.application") version "8.5.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.24" apply false
}pluginManagement {
    repositories { google(); mavenCentral(); gradlePluginPortal() }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { google(); mavenCentral() }
}
rootProject.name = "XavyAiStudio"
include(":app")