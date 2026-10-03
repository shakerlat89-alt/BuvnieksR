# Полное содержимое файлов проекта BuvnieksR

Ниже приведены все текстовые файлы, входящие в ZIP.

## `app/build.gradle`
```groovy
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}

android {
    namespace 'com.example.buvnieksr'
    compileSdk 34

    defaultConfig {
        applicationId 'com.example.buvnieksr'
        minSdk 24
        targetSdk 34
        versionCode 1
        versionName '1.0'
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = '17'
    }
}

dependencies {
    implementation 'androidx.core:core-ktx:1.12.0'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.11.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'

    implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0'
    implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.7.0'
}

```

## `app/proguard-rules.pro`
```proguard
# BuvnieksR: no custom ProGuard rules required for the current release build.

```

## `app/src/main/AndroidManifest.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        android:allowBackup="true"
        android:icon="@drawable/ic_buvnieksr"
        android:label="BuvnieksR"
        android:supportsRtl="true"
        android:theme="@style/Theme.Material3.DayNight">

        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>

        <activity
            android:name=".ui.MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>

```

## `app/src/main/java/com/example/buvnieksr/EmployeeViewModel.kt`
```kotlin
package com.example.buvnieksr

import android.app.Application
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.LiveData
import androidx.lifecycle.MutableLiveData
import com.example.buvnieksr.data.AppRepository
import com.example.buvnieksr.data.AppState
import com.example.buvnieksr.data.DashboardSummary
import com.example.buvnieksr.data.Employee
import com.example.buvnieksr.data.PayrollSummary
import com.example.buvnieksr.data.TransactionType
import java.text.SimpleDateFormat
import java.util.Calendar
import java.util.Locale
import java.util.UUID

class EmployeeViewModel(application: Application) : AndroidViewModel(application) {
    private val repository = AppRepository(application)
    private val _state = MutableLiveData<AppState>()
    val state: LiveData<AppState> = _state

    private val dateFormat = SimpleDateFormat("yyyy-MM-dd", Locale.US)
    private val monthFormat = SimpleDateFormat("yyyy-MM", Locale.US)

    private var selectedDate = today()
    private var selectedMonth = selectedDate.substring(0, 7)
    private var selectedScreen = 0

    init {
        publish()
    }

    fun selectScreen(screen: Int) {
        selectedScreen = screen.coerceIn(0, 4)
        publish()
    }

    fun changeDay(offset: Int) {
        selectedDate = shiftDate(selectedDate, offset)
        publish()
    }

    fun setDate(date: String) {
        if (date.matches(Regex("\\d{4}-\\d{2}-\\d{2}"))) {
            selectedDate = date
            publish()
        }
    }

    fun changeMonth(offset: Int) {
        val date = parseMonth(selectedMonth)
        date.add(Calendar.MONTH, offset)
        selectedMonth = monthFormat.format(date.time)
        publish()
    }

    fun setMonth(month: String) {
        if (month.matches(Regex("\\d{4}-\\d{2}"))) {
            selectedMonth = month
            publish()
        }
    }

    fun addEmployee(
        name: String,
        dailyRate: Double,
        position: String,
        specialization: String,
        phone: String,
        notes: String
    ) {
        repository.employees += Employee(
            id = UUID.randomUUID().toString(),
            name = name.trim(),
            dailyRate = dailyRate,
            position = position.trim(),
            specialization = specialization.trim(),
            phone = phone.trim(),
            notes = notes.trim()
        )
        repository.saveAll()
        publish()
    }

    fun updateEmployee(employee: Employee) {
        val index = repository.employees.indexOfFirst { it.id == employee.id }
        if (index >= 0) {
            repository.employees[index] = employee
            repository.saveAll()
            publish()
        }
    }

    fun deleteEmployee(employeeId: String) {
        repository.employees.removeAll { it.id == employeeId }
        repository.transactions.removeAll { it.employeeId == employeeId }
        repository.attendance.keys.removeAll { it.endsWith("_$employeeId") }
        repository.saveAll()
        publish()
    }

    fun attendanceValue(employeeId: String, date: String = selectedDate): Double {
        return repository.attendance["${date}_$employeeId"] ?: 0.0
    }

    fun toggleAttendance(employeeId: String, date: String = selectedDate) {
        val employee = repository.employees.firstOrNull { it.id == employeeId } ?: return
        val current = attendanceValue(employeeId, date)
        val next = when (current) {
            0.0 -> 1.0
            1.0 -> 0.5
            else -> 0.0
        }
        repository.setAttendance(employeeId, date, next, employee.dailyRate)
        publish()
    }

    fun markAllPresent() {
        repository.employees.forEach { employee ->
            repository.setAttendance(employee.id, selectedDate, 1.0, employee.dailyRate)
        }
        publish()
    }

    fun addTransaction(
        employeeId: String,
        amount: Double,
        date: String,
        type: TransactionType,
        note: String
    ) {
        repository.transactions += AppRepository.TransactionRecord(
            id = UUID.randomUUID().toString(),
            employeeId = employeeId,
            date = date,
            amount = amount,
            type = type,
            note = note.trim()
        )
        repository.saveAll()
        publish()
    }

    fun deleteTransaction(transactionId: String) {
        repository.transactions.removeAll { it.id == transactionId }
        repository.saveAll()
        publish()
    }

    fun transactionsForMonth(employeeId: String, month: String = selectedMonth): List<AppRepository.TransactionRecord> =
        repository.transactions
            .filter { it.employeeId == employeeId && it.date.startsWith(month) }
            .sortedByDescending { it.date }

    fun employeeById(id: String): Employee? = repository.employees.firstOrNull { it.id == id }

    fun language(): String = repository.language

    fun setLanguage(language: String) {
        repository.language = language
        repository.saveAll()
        publish()
    }

    fun isDarkTheme(): Boolean = repository.darkTheme

    fun setDarkTheme(enabled: Boolean) {
        repository.darkTheme = enabled
        repository.saveAll()
        publish()
    }

    fun exportBackup(): String = repository.exportBackup()

    fun importBackup(json: String) {
        runCatching {
            repository.importBackup(json)
            selectedDate = today()
            selectedMonth = selectedDate.substring(0, 7)
            publish()
        }.getOrElse { throw IllegalArgumentException("Invalid backup", it) }
    }

    fun exportCsv(): String {
        val rows = payroll().joinToString("\n") { summary ->
            listOf(
                csv(summary.employee.name),
                csv(formatMoney(summary.employee.dailyRate)),
                csv(formatDays(summary.workedDays)),
                csv(formatMoney(summary.earned)),
                csv(formatMoney(summary.advances)),
                csv(formatMoney(summary.payments)),
                csv(formatMoney(summary.fines)),
                csv(formatMoney(summary.toPay))
            ).joinToString(";")
        }
        val header = listOf("Сотрудник", "Ставка", "Дни", "Начислено", "Авансы", "Зарплата", "Штрафы", "К выплате")
            .joinToString(";")
        return "\uFEFF$header\n$rows"
    }

    private fun publish() {
        val payroll = payroll()
        val todayWorkers = repository.employees.count { attendanceValue(it.id) > 0.0 }
        val todayFund = repository.employees.sumOf {
            val factor = attendanceValue(it.id)
            factor * repository.rateSnapshot(it.id, selectedDate, it.dailyRate)
        }
        val monthEarned = payroll.sumOf { it.earned }
        val monthPaid = payroll.sumOf { it.advances + it.payments }
        val monthFines = payroll.sumOf { it.fines }
        val monthToPay = payroll.sumOf { it.toPay }
        val summary = DashboardSummary(
            todayFund = todayFund,
            workersToday = todayWorkers,
            monthFund = monthEarned,
            monthEarned = monthEarned,
            monthPaid = monthPaid,
            monthFines = monthFines,
            monthToPay = monthToPay,
            totalEmployees = repository.employees.size,
            operationCount = repository.transactions.size
        )
        _state.value = AppState(
            employees = repository.employees.toList(),
            payroll = payroll,
            dashboard = summary,
            selectedDate = selectedDate,
            selectedMonth = selectedMonth,
            selectedScreen = selectedScreen,
            language = repository.language,
            darkTheme = repository.darkTheme
        )
    }

    private fun payroll(): List<PayrollSummary> {
        return repository.employees.map { employee ->
            var days = 0.0
            var earned = 0.0
            repository.attendance.forEach { (key, value) ->
                if (key.startsWith("$selectedMonth-") && key.endsWith("_${employee.id}")) {
                    if (value > 0) {
                        days += value
                        earned += value * repository.rateSnapshotFromStoredKey(key, employee.dailyRate)
                    }
                }
            }
            var advances = 0.0
            var payments = 0.0
            var fines = 0.0
            repository.transactions
                .filter { it.employeeId == employee.id && it.date.startsWith(selectedMonth) }
                .forEach { tx ->
                    when (tx.type) {
                        TransactionType.ADVANCE -> advances += tx.amount
                        TransactionType.PAYMENT -> payments += tx.amount
                        TransactionType.FINE -> fines += tx.amount
                    }
                }
            PayrollSummary(
                employee = employee,
                workedDays = days,
                earned = earned,
                advances = advances,
                payments = payments,
                fines = fines
            )
        }
    }

    private fun AppRepository.rateSnapshotFromStoredKey(key: String, fallback: Double): Double {
        val separator = key.lastIndexOf('_')
        if (separator <= 0) return fallback
        val date = key.substring(0, separator)
        val employeeId = key.substring(separator + 1)
        return rateSnapshot(employeeId, date, fallback)
    }

    private fun today(): String = dateFormat.format(Calendar.getInstance().time)

    private fun shiftDate(value: String, offset: Int): String {
        val calendar = Calendar.getInstance().apply { time = dateFormat.parse(value)!! }
        calendar.add(Calendar.DAY_OF_MONTH, offset)
        return dateFormat.format(calendar.time)
    }

    private fun parseMonth(value: String): Calendar {
        return Calendar.getInstance().apply { time = monthFormat.parse(value)!! }
    }

    private fun formatMoney(value: Double): String = String.format(Locale.getDefault(), "%.2f €", value)
    private fun formatDays(value: Double): String = if (value % 1.0 == 0.0) value.toInt().toString() else String.format(Locale.getDefault(), "%.1f", value)
    private fun csv(value: String): String = "\"${value.replace("\"", "\"\"")}\""
}

```

## `app/src/main/java/com/example/buvnieksr/data/AppRepository.kt`
```kotlin
package com.example.buvnieksr.data

import android.content.Context
import org.json.JSONArray
import org.json.JSONObject

class AppRepository(context: Context) {
    private val prefs = context.getSharedPreferences("buvnieksr_data", Context.MODE_PRIVATE)

    var employees: MutableList<Employee> = loadEmployees()
        private set
    var attendance: MutableMap<String, Double> = loadAttendance()
        private set
    var transactions: MutableList<TransactionRecord> = loadTransactions()
        private set
    var language: String = prefs.getString("language", "ru") ?: "ru"
    var darkTheme: Boolean = prefs.getBoolean("dark_theme", false)

    data class TransactionRecord(
        val id: String,
        val employeeId: String,
        val date: String,
        val amount: Double,
        val type: TransactionType,
        val note: String
    )

    fun saveAll() {
        prefs.edit()
            .putString("employees", employeesJson(employees).toString())
            .putString("attendance", JSONObject(attendance as Map<*, *>).toString())
            .putString("transactions", transactionsJson(transactions).toString())
            .putString("language", language)
            .putBoolean("dark_theme", darkTheme)
            .apply()
    }

    fun setAttendance(employeeId: String, date: String, value: Double, rate: Double) {
        val key = "${date}_${employeeId}"
        if (value <= 0.0) {
            attendance.remove(key)
        } else {
            attendance[key] = value
            prefs.edit().putString("rate_$key", rate.toString()).apply()
        }
        saveAll()
    }

    fun rateSnapshot(employeeId: String, date: String, fallback: Double): Double {
        val key = "rate_${date}_${employeeId}"
        return prefs.getString(key, null)?.toDoubleOrNull() ?: fallback
    }

    fun exportBackup(): String {
        return JSONObject().apply {
            put("version", 1)
            put("language", language)
            put("darkTheme", darkTheme)
            put("employees", employeesJson(employees))
            put("attendance", JSONObject(attendance as Map<*, *>))
            put("transactions", transactionsJson(transactions))
        }.toString(2)
    }

    fun importBackup(jsonText: String) {
        val root = JSONObject(jsonText)
        val importedEmployees = mutableListOf<Employee>()
        val employeesArray = root.optJSONArray("employees") ?: JSONArray()
        for (i in 0 until employeesArray.length()) {
            val item = employeesArray.getJSONObject(i)
            importedEmployees += Employee(
                id = item.optString("id"),
                name = item.optString("name"),
                dailyRate = item.optDouble("dailyRate", 0.0),
                position = item.optString("position"),
                specialization = item.optString("specialization"),
                phone = item.optString("phone"),
                notes = item.optString("notes")
            )
        }

        val importedAttendance = mutableMapOf<String, Double>()
        val attendanceObject = root.optJSONObject("attendance") ?: JSONObject()
        val iterator = attendanceObject.keys()
        while (iterator.hasNext()) {
            val key = iterator.next()
            importedAttendance[key] = attendanceObject.optDouble(key, 0.0)
        }

        val importedTransactions = mutableListOf<TransactionRecord>()
        val transactionsArray = root.optJSONArray("transactions") ?: JSONArray()
        for (i in 0 until transactionsArray.length()) {
            val item = transactionsArray.getJSONObject(i)
            val type = runCatching { TransactionType.valueOf(item.optString("type")) }
                .getOrDefault(TransactionType.ADVANCE)
            importedTransactions += TransactionRecord(
                id = item.optString("id"),
                employeeId = item.optString("employeeId"),
                date = item.optString("date"),
                amount = item.optDouble("amount", 0.0),
                type = type,
                note = item.optString("note")
            )
        }

        employees = importedEmployees
        attendance = importedAttendance
        transactions = importedTransactions
        language = root.optString("language", language)
        darkTheme = root.optBoolean("darkTheme", darkTheme)
        saveAll()
    }

    private fun loadEmployees(): MutableList<Employee> {
        val result = mutableListOf<Employee>()
        val array = runCatching {
            JSONArray(prefs.getString("employees", "[]") ?: "[]")
        }.getOrDefault(JSONArray())
        for (i in 0 until array.length()) {
            val item = array.optJSONObject(i) ?: continue
            result += Employee(
                id = item.optString("id"),
                name = item.optString("name"),
                dailyRate = item.optDouble("dailyRate", 0.0),
                position = item.optString("position"),
                specialization = item.optString("specialization"),
                phone = item.optString("phone"),
                notes = item.optString("notes")
            )
        }
        return result
    }

    private fun loadAttendance(): MutableMap<String, Double> {
        val result = mutableMapOf<String, Double>()
        val json = runCatching {
            JSONObject(prefs.getString("attendance", "{}") ?: "{}")
        }.getOrDefault(JSONObject())
        val keys = json.keys()
        while (keys.hasNext()) {
            val key = keys.next()
            result[key] = json.optDouble(key, 0.0)
        }
        return result
    }

    private fun loadTransactions(): MutableList<TransactionRecord> {
        val result = mutableListOf<TransactionRecord>()
        val array = runCatching {
            JSONArray(prefs.getString("transactions", "[]") ?: "[]")
        }.getOrDefault(JSONArray())
        for (i in 0 until array.length()) {
            val item = array.optJSONObject(i) ?: continue
            val type = runCatching { TransactionType.valueOf(item.optString("type")) }
                .getOrDefault(TransactionType.ADVANCE)
            result += TransactionRecord(
                id = item.optString("id"),
                employeeId = item.optString("employeeId"),
                date = item.optString("date"),
                amount = item.optDouble("amount", 0.0),
                type = type,
                note = item.optString("note")
            )
        }
        return result
    }

    private fun employeesJson(list: List<Employee>) = JSONArray().apply {
        list.forEach { employee ->
            put(JSONObject().apply {
                put("id", employee.id)
                put("name", employee.name)
                put("dailyRate", employee.dailyRate)
                put("position", employee.position)
                put("specialization", employee.specialization)
                put("phone", employee.phone)
                put("notes", employee.notes)
            })
        }
    }

    private fun transactionsJson(list: List<TransactionRecord>) = JSONArray().apply {
        list.forEach { tx ->
            put(JSONObject().apply {
                put("id", tx.id)
                put("employeeId", tx.employeeId)
                put("date", tx.date)
                put("amount", tx.amount)
                put("type", tx.type.name)
                put("note", tx.note)
            })
        }
    }
}

```

## `app/src/main/java/com/example/buvnieksr/data/Models.kt`
```kotlin
package com.example.buvnieksr.data

data class Employee(
    val id: String,
    val name: String,
    val dailyRate: Double,
    val position: String = "",
    val specialization: String = "",
    val phone: String = "",
    val notes: String = ""
)

enum class TransactionType {
    ADVANCE,
    PAYMENT,
    FINE
}

data class PayrollSummary(
    val employee: Employee,
    val workedDays: Double,
    val earned: Double,
    val advances: Double,
    val payments: Double,
    val fines: Double
) {
    val toPay: Double get() = earned - advances - payments - fines
}

data class DashboardSummary(
    val todayFund: Double,
    val workersToday: Int,
    val monthFund: Double,
    val monthEarned: Double,
    val monthPaid: Double,
    val monthFines: Double,
    val monthToPay: Double,
    val totalEmployees: Int,
    val operationCount: Int
)

data class AppState(
    val employees: List<Employee>,
    val payroll: List<PayrollSummary>,
    val dashboard: DashboardSummary,
    val selectedDate: String,
    val selectedMonth: String,
    val selectedScreen: Int,
    val language: String,
    val darkTheme: Boolean
)

```

## `app/src/main/java/com/example/buvnieksr/ui/MainActivity.kt`
```kotlin
package com.example.buvnieksr.ui

import android.app.DatePickerDialog
import android.content.Intent
import android.net.Uri
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.widget.GridLayout
import android.widget.LinearLayout
import android.widget.RadioGroup
import android.widget.ScrollView
import android.widget.TextView
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.appcompat.app.AppCompatDelegate
import androidx.core.content.FileProvider
import androidx.lifecycle.ViewModelProvider
import com.example.buvnieksr.EmployeeViewModel
import com.example.buvnieksr.R
import com.example.buvnieksr.data.AppRepository
import com.example.buvnieksr.data.AppState
import com.example.buvnieksr.data.Employee
import com.example.buvnieksr.data.PayrollSummary
import com.example.buvnieksr.data.TransactionType
import com.google.android.material.button.MaterialButton
import com.google.android.material.card.MaterialCardView
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import com.google.android.material.textfield.TextInputEditText
import java.io.File
import java.text.SimpleDateFormat
import java.util.Calendar
import java.util.Locale

class MainActivity : AppCompatActivity() {
    private lateinit var viewModel: EmployeeViewModel
    private lateinit var result: TextView
    private lateinit var resultDetails: TextView
    private lateinit var sectionTitle: TextView
    private lateinit var quickActions: GridLayout
    private lateinit var contentContainer: LinearLayout
    private lateinit var toolbarTitle: com.google.android.material.appbar.MaterialToolbar

    private val importBackupLauncher = registerForActivityResult(ActivityResultContracts.OpenDocument()) { uri ->
        uri ?: return@registerForActivityResult
        runCatching {
            contentResolver.openInputStream(uri)?.bufferedReader()?.use { it.readText() }
                ?: error("empty")
        }.onSuccess { json ->
            runCatching { viewModel.importBackup(json) }
                .onSuccess { toast("Резервная копия загружена") }
                .onFailure { toast("Неверный файл резервной копии") }
        }.onFailure {
            toast("Не удалось открыть файл")
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        val prefs = getSharedPreferences("buvnieksr_data", MODE_PRIVATE)
        val storedDark = prefs.getBoolean("dark_theme", false)
        AppCompatDelegate.setDefaultNightMode(
            if (storedDark) AppCompatDelegate.MODE_NIGHT_YES else AppCompatDelegate.MODE_NIGHT_NO
        )
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        result = findViewById(R.id.tvResult)
        resultDetails = findViewById(R.id.tvResultDetails)
        sectionTitle = findViewById(R.id.tvSectionTitle)
        quickActions = findViewById(R.id.gridQuickActions)
        contentContainer = findViewById(R.id.contentContainer)
        toolbarTitle = findViewById(R.id.toolbar)

        viewModel = ViewModelProvider(this)[EmployeeViewModel::class.java]
        viewModel.state.observe(this) { state -> render(state) }
    }

    private fun render(state: AppState) {
        toolbarTitle.title = "🏗 BuvnieksR"
        result.text = currency(state.dashboard.monthToPay)
        resultDetails.text = "${state.dashboard.totalEmployees} сотрудников · ${state.dashboard.operationCount} операций"
        sectionTitle.text = when (state.selectedScreen) {
            0 -> "Быстрые действия"
            1 -> "Отметки за день"
            2 -> "Расчёт зарплаты"
            3 -> "Контакты"
            else -> "Настройки"
        }
        buildQuickActions(state)
        contentContainer.removeAllViews()

        when (state.selectedScreen) {
            0 -> renderEmployees(state)
            1 -> renderAttendance(state)
            2 -> renderPayroll(state)
            3 -> renderContacts(state)
            else -> renderSettings(state)
        }
    }

    private fun buildQuickActions(state: AppState) {
        quickActions.removeAllViews()
        val actions = listOf(
            Action("👷", "Сотрудники") { viewModel.selectScreen(0) },
            Action("📅", "Табель") { viewModel.selectScreen(1) },
            Action("💶", "Зарплата") { viewModel.selectScreen(2) },
            Action("📞", "Контакты") { viewModel.selectScreen(3) },
            Action("⚙️", "Настройки") { viewModel.selectScreen(4) },
            Action("＋", "Сотрудник") { showEmployeeDialog(null) },
            Action("＋ €", "Аванс") { showTransactionPicker(TransactionType.ADVANCE) },
            Action("＋ €", "Выдать зарплату") { showTransactionPicker(TransactionType.PAYMENT) },
            Action("− €", "Штраф") { showTransactionPicker(TransactionType.FINE) }
        )
        actions.forEachIndexed { index, action ->
            val button = MaterialButton(this).apply {
                text = "${action.icon}\n${action.label}"
                isAllCaps = false
                textSize = 12f
                minHeight = dp(68)
                minimumWidth = 0
                setPadding(dp(3), dp(5), dp(3), dp(5))
                cornerRadius = dp(16)
                setOnClickListener { action.onClick() }
            }
            val params = GridLayout.LayoutParams().apply {
                width = 0
                height = GridLayout.LayoutParams.WRAP_CONTENT
                columnSpec = GridLayout.spec(index % 3, 1f)
                rowSpec = GridLayout.spec(index / 3)
                setMargins(dp(3), dp(3), dp(3), dp(3))
            }
            quickActions.addView(button, params)
        }
    }

    private fun renderEmployees(state: AppState) {
        addSectionCard(
            title = "Сводка за ${monthLabel(state.selectedMonth)}",
            subtitle = "Фонд за день: ${currency(state.dashboard.todayFund)} · Работает сегодня: ${state.dashboard.workersToday}/${state.dashboard.totalEmployees}",
            value = currency(state.dashboard.monthToPay),
            accent = true
        )
        if (state.employees.isEmpty()) {
            addEmpty("Список пуст — добавьте первого сотрудника.")
            return
        }
        state.payroll.forEach { summary -> addEmployeeItem(summary) }
    }

    private fun renderAttendance(state: AppState) {
        addDateNavigator(state.selectedDate)
        MaterialButton(this).apply {
            text = "✔ Отметить всех"
            isAllCaps = false
            setOnClickListener { viewModel.markAllPresent() }
        }.also { contentContainer.addView(it, fullWidthParams()) }

        if (state.employees.isEmpty()) {
            addEmpty("Список сотрудников пуст.")
            return
        }
        state.employees.forEach { employee ->
            val card = LayoutInflater.from(this).inflate(R.layout.item_employee, contentContainer, false) as MaterialCardView
            val avatar = card.findViewById<TextView>(R.id.tvAvatar)
            val name = card.findViewById<TextView>(R.id.tvName)
            val summary = card.findViewById<TextView>(R.id.tvSummary)
            val phone = card.findViewById<TextView>(R.id.tvPhone)
            val action = card.findViewById<MaterialButton>(R.id.btnItemAction)
            avatar.text = employee.name.firstOrNull()?.uppercase() ?: "?"
            name.text = employee.name
            val value = viewModel.attendanceValue(employee.id, state.selectedDate)
            summary.text = when (value) {
                1.0 -> "✔ Был · 1 день"
                0.5 -> "½ Полдня"
                else -> "○ Не отмечен"
            }
            phone.text = "${currency(employee.dailyRate)} / день"
            action.text = "${if (value == 0.0) "Отметить" else "Сбросить"}"
            action.setOnClickListener { viewModel.toggleAttendance(employee.id) }
            card.setOnClickListener { showEmployeeDialog(employee) }
            contentContainer.addView(card)
        }
    }

    private fun renderPayroll(state: AppState) {
        addMonthNavigator(state.selectedMonth)
        val d = state.dashboard
        addSectionCard("Фонд за месяц", "Начислено ${currency(d.monthEarned)}", currency(d.monthToPay), true)
        addStatsRow(
            listOf(
                "Начислено" to currency(d.monthEarned),
                "Выдано" to currency(d.monthPaid),
                "Штрафы" to currency(d.monthFines)
            )
        )
        if (state.payroll.isEmpty()) {
            addEmpty("Нет сотрудников для расчёта.")
            return
        }
        state.payroll.forEach { summary -> addPayrollCard(summary) }
    }

    private fun renderContacts(state: AppState) {
        if (state.employees.isEmpty()) {
            addEmpty("Добавьте сотрудников, чтобы видеть контакты.")
            return
        }
        state.employees.forEach { employee ->
            val card = LayoutInflater.from(this).inflate(R.layout.item_employee, contentContainer, false) as MaterialCardView
            val avatar = card.findViewById<TextView>(R.id.tvAvatar)
            val name = card.findViewById<TextView>(R.id.tvName)
            val summary = card.findViewById<TextView>(R.id.tvSummary)
            val phone = card.findViewById<TextView>(R.id.tvPhone)
            val action = card.findViewById<MaterialButton>(R.id.btnItemAction)
            avatar.text = employee.name.firstOrNull()?.uppercase() ?: "?"
            name.text = employee.name
            summary.text = employee.position.ifBlank { "Сотрудник" }
            phone.text = employee.phone.ifBlank { "Номер не указан" }
            action.text = "Позвонить"
            action.setOnClickListener { dial(employee.phone) }
            contentContainer.addView(card)
        }
    }

    private fun renderSettings(state: AppState) {
        addSimpleCard("Тема", "Выберите оформление приложения") {
            val row = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL }
            row.addView(settingsButton("Светлая", !state.darkTheme) {
                viewModel.setDarkTheme(false)
                AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_NO)
                recreate()
            }, weightParams())
            row.addView(settingsButton("Тёмная", state.darkTheme) {
                viewModel.setDarkTheme(true)
                AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_YES)
                recreate()
            }, weightParams())
            row
        }
        addSimpleCard("Данные", "Локальное хранение на устройстве") {
            val row = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL }
            row.addView(settingsButton("Экспорт CSV", false) { shareTextFile("BuvnieksR-${state.selectedMonth}.csv", viewModel.exportCsv()) }, weightParams())
            row.addView(settingsButton("Копия JSON", false) { shareTextFile("BuvnieksR-backup.json", viewModel.exportBackup()) }, weightParams())
            row
        }
        addSimpleCard("Восстановление", "Импорт ранее сохранённой резервной копии") {
            MaterialButton(this).apply {
                text = "Загрузить JSON-копию"
                isAllCaps = false
                setOnClickListener { importBackupLauncher.launch(arrayOf("application/json", "text/plain")) }
            }
        }
        addSimpleCard("О приложении", "BuvnieksR 1.0") {
            TextView(this).apply {
                text = "Учёт сотрудников, табель, авансы, выплаты и штрафы. Расчёт выполняется локально без серверной базы."
                setTextAppearance(com.google.android.material.R.style.TextAppearance_Material3_BodyMedium)
            }
        }
    }

    private fun addEmployeeItem(summary: PayrollSummary) {
        val employee = summary.employee
        val card = LayoutInflater.from(this).inflate(R.layout.item_employee, contentContainer, false) as MaterialCardView
        card.findViewById<TextView>(R.id.tvAvatar).text = employee.name.firstOrNull()?.uppercase() ?: "?"
        card.findViewById<TextView>(R.id.tvName).text = employee.name
        card.findViewById<TextView>(R.id.tvSummary).text = "${formatDays(summary.workedDays)} дн. · начислено ${currency(summary.earned)}"
        card.findViewById<TextView>(R.id.tvPhone).text = "К выплате: ${currency(summary.toPay)}"
        card.findViewById<MaterialButton>(R.id.btnItemAction).apply {
            text = "Открыть"
            setOnClickListener { showEmployeeDialog(employee) }
        }
        card.setOnClickListener { showEmployeeDialog(employee) }
        contentContainer.addView(card)
    }

    private fun addPayrollCard(summary: PayrollSummary) {
        val card = MaterialCardView(this).apply {
            radius = dp(20).toFloat()
            cardElevation = dp(1).toFloat()
            setContentPadding(dp(16), dp(14), dp(16), dp(14))
            layoutParams = LinearLayout.LayoutParams(-1, -2).apply { bottomMargin = dp(10) }
        }
        val box = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL }
        val title = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL; gravity = android.view.Gravity.CENTER_VERTICAL }
        val name = TextView(this).apply {
            text = summary.employee.name
            setTextAppearance(com.google.android.material.R.style.TextAppearance_Material3_TitleMedium)
            setTypeface(typeface, android.graphics.Typeface.BOLD)
        }
        val amount = TextView(this).apply {
            text = currency(summary.toPay)
            setTextAppearance(com.google.android.material.R.style.TextAppearance_Material3_TitleLarge)
            setTypeface(typeface, android.graphics.Typeface.BOLD)
        }
        title.addView(name, LinearLayout.LayoutParams(0, -2, 1f))
        title.addView(amount)
        box.addView(title)
        box.addView(textLine("${formatDays(summary.workedDays)} дн. × ${currency(summary.employee.dailyRate)} = ${currency(summary.earned)}"))
        box.addView(textLine("Авансы: −${currency(summary.advances)} · Выплаты: −${currency(summary.payments)} · Штрафы: −${currency(summary.fines)}"))
        card.setOnClickListener { showEmployeeDialog(summary.employee) }
        card.addView(box)
        contentContainer.addView(card)
    }

    private fun showEmployeeDialog(employee: Employee?) {
        val dialogView = LayoutInflater.from(this).inflate(R.layout.dialog_employee, null)
        val name = dialogView.findViewById<TextInputEditText>(R.id.etEmployeeName)
        val rate = dialogView.findViewById<TextInputEditText>(R.id.etDailyRate)
        val position = dialogView.findViewById<TextInputEditText>(R.id.etPosition)
        val specialization = dialogView.findViewById<TextInputEditText>(R.id.etSpecialization)
        val phone = dialogView.findViewById<TextInputEditText>(R.id.etPhone)
        val notes = dialogView.findViewById<TextInputEditText>(R.id.etNotes)

        employee?.let {
            name.setText(it.name)
            rate.setText(it.dailyRate.toString())
            position.setText(it.position)
            specialization.setText(it.specialization)
            phone.setText(it.phone)
            notes.setText(it.notes)

            val content = (dialogView as? ScrollView)?.getChildAt(0) as? LinearLayout
            content?.addView(TextView(this).apply {
                text = "Операции за ${monthLabel(viewModel.state.value?.selectedMonth ?: today().substring(0, 7))}"
                setTextAppearance(com.google.android.material.R.style.TextAppearance_Material3_TitleMedium)
                setTypeface(typeface, android.graphics.Typeface.BOLD)
                setPadding(0, dp(14), 0, dp(6))
            })
            val operations = viewModel.transactionsForMonth(it.id)
            if (content != null && operations.isNotEmpty()) {
                operations.forEach { tx ->
                    val row = MaterialCardView(this).apply {
                        radius = dp(14).toFloat()
                        setContentPadding(dp(10), dp(8), dp(10), dp(8))
                        setOnClickListener {
                            MaterialAlertDialogBuilder(this@MainActivity)
                                .setTitle("Удалить операцию?")
                                .setMessage("${operationTitle(tx.type)} · ${currency(tx.amount)} · ${tx.date}")
                                .setNegativeButton("Отмена", null)
                                .setPositiveButton("Удалить") { _, _ -> viewModel.deleteTransaction(tx.id); toast("Удалено") }
                                .show()
                        }
                    }
                    row.addView(TextView(this).apply {
                        text = "${operationShortLabel(tx.type)} · ${tx.date} · ${currency(tx.amount)}" +
                            if (tx.note.isBlank()) "" else "\n${tx.note}"
                        setTextAppearance(com.google.android.material.R.style.TextAppearance_Material3_BodyMedium)
                    })
                    content.addView(row, LinearLayout.LayoutParams(-1, -2).apply { bottomMargin = dp(6) })
                }
            } else if (content != null) {
                content.addView(TextView(this).apply {
                    text = "Операций за выбранный месяц нет."
                    setTextAppearance(com.google.android.material.R.style.TextAppearance_Material3_BodySmall)
                    setPadding(0, 0, 0, dp(6))
                })
            }
        }

        val dialog = MaterialAlertDialogBuilder(this)
            .setTitle(if (employee == null) "Добавить сотрудника" else "Данные сотрудника")
            .setView(dialogView)
            .setNegativeButton("Отмена", null)
            .setPositiveButton("Сохранить", null)
            .apply {
                if (employee != null) {
                    setNeutralButton("Удалить", null)
                }
            }
            .create()

        dialog.setOnShowListener {
            dialog.getButton(android.content.DialogInterface.BUTTON_POSITIVE).setOnClickListener {
                val parsedRate = rate.text?.toString()?.replace(',', '.')?.toDoubleOrNull()
                if (name.text.isNullOrBlank() || parsedRate == null || parsedRate < 0.0) {
                    toast("Укажите имя и корректную ставку")
                    return@setOnClickListener
                }
                val updated = Employee(
                    id = employee?.id ?: java.util.UUID.randomUUID().toString(),
                    name = name.text.toString().trim(),
                    dailyRate = parsedRate,
                    position = position.text?.toString().orEmpty().trim(),
                    specialization = specialization.text?.toString().orEmpty().trim(),
                    phone = phone.text?.toString().orEmpty().trim(),
                    notes = notes.text?.toString().orEmpty().trim()
                )
                if (employee == null) {
                    viewModel.addEmployee(updated.name, updated.dailyRate, updated.position, updated.specialization, updated.phone, updated.notes)
                } else {
                    viewModel.updateEmployee(updated)
                }
                dialog.dismiss()
                toast("Сохранено")
            }
            employee?.let {
                dialog.getButton(android.content.DialogInterface.BUTTON_NEUTRAL).setOnClickListener {
                    MaterialAlertDialogBuilder(this)
                        .setTitle("Удалить сотрудника?")
                        .setMessage("Все связанные отметки и операции тоже будут удалены.")
                        .setNegativeButton("Отмена", null)
                        .setPositiveButton("Удалить") { _, _ -> viewModel.deleteEmployee(it.id); dialog.dismiss(); toast("Удалено") }
                        .show()
                }
            }
        }
        dialog.show()
    }

    private fun showTransactionPicker(type: TransactionType) {
        val employees = viewModel.state.value?.employees.orEmpty()
        if (employees.isEmpty()) {
            toast("Сначала добавьте сотрудника")
            return
        }
        val names = employees.map { it.name }.toTypedArray()
        MaterialAlertDialogBuilder(this)
            .setTitle(operationTitle(type))
            .setItems(names) { _, which -> showTransactionDialog(employees[which], type) }
            .setNegativeButton("Отмена", null)
            .show()
    }

    private fun showTransactionDialog(employee: Employee, presetType: TransactionType = TransactionType.ADVANCE) {
        val dialogView = LayoutInflater.from(this).inflate(R.layout.dialog_transaction, null)
        val group = dialogView.findViewById<RadioGroup>(R.id.rgOperationType)
        val amount = dialogView.findViewById<TextInputEditText>(R.id.etAmount)
        val date = dialogView.findViewById<TextInputEditText>(R.id.etDate)
        val comment = dialogView.findViewById<TextInputEditText>(R.id.etComment)

        when (presetType) {
            TransactionType.ADVANCE -> group.check(R.id.rbAdvance)
            TransactionType.PAYMENT -> group.check(R.id.rbPayment)
            TransactionType.FINE -> group.check(R.id.rbFine)
        }
        date.setText(viewModel.state.value?.selectedDate ?: today())
        date.setOnClickListener { pickDate(date) }

        val dialog = MaterialAlertDialogBuilder(this)
            .setTitle("Операция · ${employee.name}")
            .setView(dialogView)
            .setNegativeButton("Отмена", null)
            .setPositiveButton("Сохранить", null)
            .create()

        dialog.setOnShowListener {
            dialog.getButton(android.content.DialogInterface.BUTTON_POSITIVE).setOnClickListener {
                val value = amount.text?.toString()?.replace(',', '.')?.toDoubleOrNull()
                val dateValue = date.text?.toString().orEmpty()
                if (value == null || value <= 0.0 || !dateValue.matches(Regex("\\d{4}-\\d{2}-\\d{2}"))) {
                    toast("Укажите сумму и дату")
                    return@setOnClickListener
                }
                val txType = when (group.checkedRadioButtonId) {
                    R.id.rbPayment -> TransactionType.PAYMENT
                    R.id.rbFine -> TransactionType.FINE
                    else -> TransactionType.ADVANCE
                }
                viewModel.addTransaction(employee.id, value, dateValue, txType, comment.text?.toString().orEmpty())
                dialog.dismiss()
                toast("Сохранено")
            }
        }
        dialog.show()
    }

    private fun addDateNavigator(date: String) {
        val row = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL; gravity = android.view.Gravity.CENTER_VERTICAL }
        row.addView(navButton("‹") { viewModel.changeDay(-1) }, navButtonParams())
        row.addView(MaterialButton(this).apply {
            text = "${prettyDate(date)}"
            isAllCaps = false
            setOnClickListener { pickMainDate() }
        }, navCenterParams())
        row.addView(navButton("›") { viewModel.changeDay(1) }, navButtonParams())
        contentContainer.addView(row)
    }

    private fun addMonthNavigator(month: String) {
        val row = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL; gravity = android.view.Gravity.CENTER_VERTICAL }
        row.addView(navButton("‹") { viewModel.changeMonth(-1) }, navButtonParams())
        row.addView(MaterialButton(this).apply {
            text = monthLabel(month)
            isAllCaps = false
            setOnClickListener { pickMonth() }
        }, navCenterParams())
        row.addView(navButton("›") { viewModel.changeMonth(1) }, navButtonParams())
        contentContainer.addView(row)
    }

    private fun pickMainDate() {
        val current = parseDate(viewModel.state.value?.selectedDate ?: today())
        DatePickerDialog(this, { _, year, month, day ->
            val selected = Calendar.getInstance().apply { set(year, month, day) }
            viewModel.setDate(SimpleDateFormat("yyyy-MM-dd", Locale.US).format(selected.time))
        }, current.get(Calendar.YEAR), current.get(Calendar.MONTH), current.get(Calendar.DAY_OF_MONTH)).show()
    }

    private fun pickDate(target: TextInputEditText) {
        val current = parseDate(target.text?.toString().orEmpty().ifBlank { today() })
        DatePickerDialog(this, { _, year, month, day ->
            val selected = Calendar.getInstance().apply { set(year, month, day) }
            target.setText(SimpleDateFormat("yyyy-MM-dd", Locale.US).format(selected.time))
        }, current.get(Calendar.YEAR), current.get(Calendar.MONTH), current.get(Calendar.DAY_OF_MONTH)).show()
    }

    private fun pickMonth() {
        val current = parseMonth(viewModel.state.value?.selectedMonth ?: today().substring(0, 7))
        DatePickerDialog(this, { _, year, month, _ ->
            viewModel.setMonth(String.format(Locale.US, "%04d-%02d", year, month + 1))
        }, current.get(Calendar.YEAR), current.get(Calendar.MONTH), 1).show()
    }

    private fun addSectionCard(title: String, subtitle: String, value: String, accent: Boolean) {
        val card = MaterialCardView(this).apply {
            radius = dp(20).toFloat()
            setContentPadding(dp(16), dp(14), dp(16), dp(14))
            layoutParams = LinearLayout.LayoutParams(-1, -2).apply { bottomMargin = dp(10) }
        }
        val box = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL }
        box.addView(textLine(title, true))
        box.addView(textLine(subtitle))
        box.addView(TextView(this).apply {
            text = value
            setTextAppearance(if (accent) com.google.android.material.R.style.TextAppearance_Material3_DisplaySmall else com.google.android.material.R.style.TextAppearance_Material3.HeadlineMedium)
            setTypeface(typeface, android.graphics.Typeface.BOLD)
            setPadding(0, dp(5), 0, 0)
        })
        card.addView(box)
        contentContainer.addView(card)
    }

    private fun addStatsRow(items: List<Pair<String, String>>) {
        val row = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL }
        items.forEach { (label, value) ->
            val card = MaterialCardView(this).apply {
                radius = dp(16).toFloat()
                setContentPadding(dp(10), dp(10), dp(10), dp(10))
            }
            val box = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL }
            box.addView(textLine(label))
            box.addView(TextView(this).apply {
                text = value
                setTextAppearance(com.google.android.material.R.style.TextAppearance_Material3_TitleMedium)
                setTypeface(typeface, android.graphics.Typeface.BOLD)
            })
            card.addView(box)
            row.addView(card, weightParams(1f))
        }
        row.showDividers = LinearLayout.SHOW_DIVIDER_MIDDLE
        contentContainer.addView(row, LinearLayout.LayoutParams(-1, -2).apply { bottomMargin = dp(10) })
    }

    private fun addSimpleCard(title: String, subtitle: String, contentFactory: () -> View) {
        val card = MaterialCardView(this).apply {
            radius = dp(20).toFloat()
            setContentPadding(dp(16), dp(14), dp(16), dp(14))
            layoutParams = LinearLayout.LayoutParams(-1, -2).apply { bottomMargin = dp(10) }
        }
        val box = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL }
        box.addView(textLine(title, true))
        box.addView(textLine(subtitle))
        val content = contentFactory()
        box.addView(content, LinearLayout.LayoutParams(-1, -2).apply { topMargin = dp(10) })
        card.addView(box)
        contentContainer.addView(card)
    }

    private fun addEmpty(text: String) {
        val tv = TextView(this).apply {
            text = text
            gravity = android.view.Gravity.CENTER
            setPadding(dp(24), dp(32), dp(24), dp(32))
            setTextAppearance(com.google.android.material.R.style.TextAppearance_Material3_BodyLarge)
        }
        contentContainer.addView(tv, fullWidthParams())
    }

    private fun textLine(text: String, title: Boolean = false): TextView = TextView(this).apply {
        this.text = text
        setTextAppearance(if (title) com.google.android.material.R.style.TextAppearance_Material3.TitleMedium else com.google.android.material.R.style.TextAppearance_Material3.BodySmall)
    }

    private fun settingsButton(label: String, selected: Boolean, onClick: () -> Unit): MaterialButton = MaterialButton(this).apply {
        text = label
        isAllCaps = false
        isChecked = selected
        setOnClickListener { onClick() }
    }

    private fun navButton(label: String, action: () -> Unit) = MaterialButton(this).apply {
        text = label
        isAllCaps = false
        textSize = 22f
        minHeight = dp(48)
        setOnClickListener { action() }
    }

    private fun dial(phone: String) {
        val cleaned = phone.filter { it.isDigit() || it == '+' }
        if (cleaned.length < 3) {
            toast("Номер телефона не указан")
            return
        }
        startActivity(Intent(Intent.ACTION_DIAL, Uri.parse("tel:$cleaned")))
    }

    private fun shareTextFile(fileName: String, content: String) {
        val sharedDir = File(filesDir, "shared").apply { mkdirs() }
        val file = File(sharedDir, fileName)
        file.writeText(content, Charsets.UTF_8)
        val uri = FileProvider.getUriForFile(this, "${packageName}.fileprovider", file)
        startActivity(Intent.createChooser(Intent(Intent.ACTION_SEND).apply {
            type = if (fileName.endsWith(".csv")) "text/csv" else "application/json"
            putExtra(Intent.EXTRA_STREAM, uri)
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }, "Поделиться файлом"))
    }

    private fun operationShortLabel(type: TransactionType): String = when (type) {
        TransactionType.ADVANCE -> "Аванс"
        TransactionType.PAYMENT -> "Зарплата"
        TransactionType.FINE -> "Штраф"
    }

    private fun operationTitle(type: TransactionType): String = when (type) {
        TransactionType.ADVANCE -> "Выберите сотрудника для аванса"
        TransactionType.PAYMENT -> "Выберите сотрудника для выплаты"
        TransactionType.FINE -> "Выберите сотрудника для штрафа"
    }

    private fun prettyDate(value: String): String = runCatching {
        SimpleDateFormat("d MMMM", Locale("ru")).format(SimpleDateFormat("yyyy-MM-dd", Locale.US).parse(value)!!)
    }.getOrDefault(value)

    private fun monthLabel(value: String): String = runCatching {
        SimpleDateFormat("LLLL yyyy", Locale("ru")).format(SimpleDateFormat("yyyy-MM", Locale.US).parse(value)!!)
            .replaceFirstChar { it.titlecase(Locale("ru")) }
    }.getOrDefault(value)

    private fun parseDate(value: String): Calendar = Calendar.getInstance().apply {
        time = SimpleDateFormat("yyyy-MM-dd", Locale.US).parse(value) ?: time
    }

    private fun parseMonth(value: String): Calendar = Calendar.getInstance().apply {
        time = SimpleDateFormat("yyyy-MM", Locale.US).parse(value) ?: time
    }

    private fun today(): String = SimpleDateFormat("yyyy-MM-dd", Locale.US).format(Calendar.getInstance().time)

    private fun currency(value: Double): String = String.format(Locale.getDefault(), "%.2f €", value)

    private fun formatDays(value: Double): String = if (value % 1.0 == 0.0) {
        value.toInt().toString()
    } else {
        String.format(Locale.getDefault(), "%.1f", value)
    }

    private fun toast(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }

    private fun dp(value: Int): Int = (value * resources.displayMetrics.density).toInt()

    private fun fullWidthParams(): LinearLayout.LayoutParams = LinearLayout.LayoutParams(-1, -2).apply { bottomMargin = dp(10) }
    private fun navButtonParams(): LinearLayout.LayoutParams = LinearLayout.LayoutParams(dp(52), -2)
    private fun navCenterParams(): LinearLayout.LayoutParams = LinearLayout.LayoutParams(0, -2, 1f)
    private fun weightParams(weight: Float = 1f): LinearLayout.LayoutParams = LinearLayout.LayoutParams(0, -2, weight).apply { marginEnd = dp(6) }

    private data class Action(val icon: String, val label: String, val onClick: () -> Unit)
}

```

## `app/src/main/res/drawable/ic_avatar_bg.xml`
```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="rectangle">
    <corners android:radius="16dp" />
    <solid android:color="#4F46E5" />
</shape>

```

## `app/src/main/res/drawable/ic_buvnieksr.xml`
```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="48dp"
    android:height="48dp"
    android:viewportWidth="48"
    android:viewportHeight="48">
    <path
        android:fillColor="#4F46E5"
        android:pathData="M24,3A21,21 0,1 0,24 45A21,21 0,1 0,24 3" />
    <path
        android:fillColor="#FFFFFF"
        android:pathData="M13,34V18h5v16zM20,34V14h5v20zM27,34V10h5v24zM34,34V20h2v14z" />
    <path
        android:fillColor="#FFFFFF"
        android:pathData="M10,36h28v3h-28z" />
</vector>

```

## `app/src/main/res/layout/activity_main.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="?android:attr/colorBackground">

    <com.google.android.material.appbar.MaterialToolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:minHeight="64dp"
        app:title="BuvnieksR"
        app:titleTextAppearance="@style/TextAppearance.Material3.HeadlineSmall" />

    <ScrollView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:fillViewport="true">

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:padding="16dp">

            <com.google.android.material.card.MaterialCardView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                app:cardCornerRadius="24dp"
                app:cardUseCompatPadding="true">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:padding="18dp">

                    <TextView
                        android:id="@+id/tvResultLabel"
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="Результат"
                        android:textAppearance="@style/TextAppearance.Material3.LabelLarge" />

                    <TextView
                        android:id="@+id/tvResult"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:layout_marginTop="2dp"
                        android:text="0,00 €"
                        android:textAppearance="@style/TextAppearance.Material3.DisplaySmall"
                        android:textStyle="bold" />

                    <TextView
                        android:id="@+id/tvResultDetails"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:layout_marginTop="4dp"
                        android:text="0 сотрудников · 0 операций"
                        android:textAppearance="@style/TextAppearance.Material3.BodyMedium" />
                </LinearLayout>
            </com.google.android.material.card.MaterialCardView>

            <TextView
                android:id="@+id/tvSectionTitle"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="12dp"
                android:text="Быстрые действия"
                android:textAppearance="@style/TextAppearance.Material3.TitleLarge" />

            <GridLayout
                android:id="@+id/gridQuickActions"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:columnCount="3"
                android:rowCount="3"
                android:alignmentMode="alignMargins"
                android:useDefaultMargins="true" />

            <LinearLayout
                android:id="@+id/contentContainer"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="6dp"
                android:orientation="vertical" />
        </LinearLayout>
    </ScrollView>
</LinearLayout>

```

## `app/src/main/res/layout/dialog_employee.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="4dp">

        <com.google.android.material.textfield.TextInputLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content">
            <com.google.android.material.textfield.TextInputEditText
                android:id="@+id/etEmployeeName"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="Имя и фамилия"
                android:inputType="textCapWords|textPersonName" />
        </com.google.android.material.textfield.TextInputLayout>

        <com.google.android.material.textfield.TextInputLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp">
            <com.google.android.material.textfield.TextInputEditText
                android:id="@+id/etDailyRate"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="Ставка в день (€)"
                android:inputType="numberDecimal" />
        </com.google.android.material.textfield.TextInputLayout>

        <com.google.android.material.textfield.TextInputLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp">
            <com.google.android.material.textfield.TextInputEditText
                android:id="@+id/etPosition"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="Должность"
                android:inputType="textCapSentences" />
        </com.google.android.material.textfield.TextInputLayout>

        <com.google.android.material.textfield.TextInputLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp">
            <com.google.android.material.textfield.TextInputEditText
                android:id="@+id/etSpecialization"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="Специализация"
                android:inputType="textCapSentences" />
        </com.google.android.material.textfield.TextInputLayout>

        <com.google.android.material.textfield.TextInputLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp">
            <com.google.android.material.textfield.TextInputEditText
                android:id="@+id/etPhone"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="Телефон"
                android:inputType="phone" />
        </com.google.android.material.textfield.TextInputLayout>

        <com.google.android.material.textfield.TextInputLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp">
            <com.google.android.material.textfield.TextInputEditText
                android:id="@+id/etNotes"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="Заметки"
                android:gravity="top"
                android:minLines="3"
                android:inputType="textCapSentences|textMultiLine" />
        </com.google.android.material.textfield.TextInputLayout>
    </LinearLayout>
</ScrollView>

```

## `app/src/main/res/layout/dialog_transaction.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="4dp">

    <RadioGroup
        android:id="@+id/rgOperationType"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal">

        <com.google.android.material.radiobutton.MaterialRadioButton
            android:id="@+id/rbAdvance"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Аванс"
            android:checked="true" />

        <com.google.android.material.radiobutton.MaterialRadioButton
            android:id="@+id/rbPayment"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Зарплата" />

        <com.google.android.material.radiobutton.MaterialRadioButton
            android:id="@+id/rbFine"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Штраф" />
    </RadioGroup>

    <com.google.android.material.textfield.TextInputLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="10dp">
        <com.google.android.material.textfield.TextInputEditText
            android:id="@+id/etAmount"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:hint="Сумма (€)"
            android:inputType="numberDecimal" />
    </com.google.android.material.textfield.TextInputLayout>

    <com.google.android.material.textfield.TextInputLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="10dp">
        <com.google.android.material.textfield.TextInputEditText
            android:id="@+id/etDate"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:hint="Дата"
            android:inputType="date" />
    </com.google.android.material.textfield.TextInputLayout>

    <com.google.android.material.textfield.TextInputLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="10dp">
        <com.google.android.material.textfield.TextInputEditText
            android:id="@+id/etComment"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:hint="Комментарий"
            android:inputType="textCapSentences|textMultiLine" />
    </com.google.android.material.textfield.TextInputLayout>
</LinearLayout>

```

## `app/src/main/res/layout/item_employee.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<com.google.android.material.card.MaterialCardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginBottom="10dp"
    app:cardCornerRadius="18dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center_vertical"
        android:orientation="horizontal"
        android:padding="12dp">

        <TextView
            android:id="@+id/tvAvatar"
            android:layout_width="46dp"
            android:layout_height="46dp"
            android:background="@drawable/ic_avatar_bg"
            android:gravity="center"
            android:text="И"
            android:textAppearance="@style/TextAppearance.Material3.TitleLarge"
            android:textColor="@android:color/white"
            android:textStyle="bold" />

        <LinearLayout
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:layout_marginStart="12dp"
            android:orientation="vertical">

            <TextView
                android:id="@+id/tvName"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:ellipsize="end"
                android:maxLines="1"
                android:textAppearance="@style/TextAppearance.Material3.TitleMedium"
                android:textStyle="bold" />

            <TextView
                android:id="@+id/tvSummary"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="2dp"
                android:textAppearance="@style/TextAppearance.Material3.BodySmall" />

            <TextView
                android:id="@+id/tvPhone"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:textAppearance="@style/TextAppearance.Material3.BodySmall" />
        </LinearLayout>

        <com.google.android.material.button.MaterialButton
            android:id="@+id/btnItemAction"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:minWidth="0dp"
            android:text="Открыть"
            android:textSize="12sp"
            app:cornerRadius="14dp" />
    </LinearLayout>
</com.google.android.material.card.MaterialCardView>

```

## `app/src/main/res/layout/item_operation.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<com.google.android.material.card.MaterialCardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginBottom="8dp"
    app:cardCornerRadius="16dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center_vertical"
        android:orientation="horizontal"
        android:padding="12dp">

        <TextView
            android:id="@+id/tvOperationType"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Аванс"
            android:textAppearance="@style/TextAppearance.Material3.LabelLarge" />

        <LinearLayout
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:layout_marginStart="12dp"
            android:orientation="vertical">
            <TextView
                android:id="@+id/tvOperationDate"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:textAppearance="@style/TextAppearance.Material3.BodySmall" />
            <TextView
                android:id="@+id/tvOperationNote"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:ellipsize="end"
                android:maxLines="1"
                android:textAppearance="@style/TextAppearance.Material3.BodySmall" />
        </LinearLayout>

        <TextView
            android:id="@+id/tvOperationAmount"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.Material3.TitleMedium"
            android:textStyle="bold" />
    </LinearLayout>
</com.google.android.material.card.MaterialCardView>

```

## `app/src/main/res/values/strings.xml`
```xml
<resources>
    <string name="app_name">BuvnieksR</string>
    <string name="navigation">Разделы</string>
    <string name="result">Результат</string>
    <string name="add_employee">Добавить сотрудника</string>
    <string name="save">Сохранить</string>
    <string name="cancel">Отмена</string>
    <string name="delete">Удалить</string>
    <string name="employee_name">Имя и фамилия</string>
    <string name="daily_rate">Ставка в день (€)</string>
    <string name="position">Должность</string>
    <string name="specialization">Специализация</string>
    <string name="phone">Телефон</string>
    <string name="notes">Заметки</string>
    <string name="attendance">Табель</string>
    <string name="payroll">Зарплата</string>
    <string name="contacts">Контакты</string>
    <string name="employees">Сотрудники</string>
    <string name="settings">Настройки</string>
    <string name="advance">Аванс</string>
    <string name="payment">Зарплата</string>
    <string name="fine">Штраф</string>
    <string name="today_fund">Фонд за день</string>
    <string name="month_fund">Фонд за месяц</string>
    <string name="earned">Начислено</string>
    <string name="paid">Выдано</string>
    <string name="fines">Штрафы</string>
    <string name="to_pay">К выплате</string>
    <string name="worked_days">Отработано дней</string>
    <string name="mark_all">Отметить всех</string>
    <string name="present">Был</string>
    <string name="half_day">Полдня</string>
    <string name="absent">Не был</string>
    <string name="quick_actions">Быстрые действия</string>
    <string name="monthly_summary">Сводка за месяц</string>
    <string name="export_csv">Экспорт CSV</string>
    <string name="backup_json">Резервная копия JSON</string>
    <string name="appearance">Оформление</string>
    <string name="light">Светлая</string>
    <string name="dark">Тёмная</string>
    <string name="language">Язык</string>
    <string name="russian">Русский</string>
    <string name="latvian">Latviešu</string>
    <string name="select_date">Выберите дату</string>
    <string name="select_month">Выберите месяц</string>
    <string name="operation">Операция</string>
    <string name="amount">Сумма (€)</string>
    <string name="comment">Комментарий</string>
    <string name="delete_confirmation">Удалить запись?</string>
    <string name="empty_employees">Список пуст. Добавьте первого сотрудника.</string>
    <string name="empty_operations">Операций за этот месяц нет.</string>
    <string name="no_phone">У сотрудника нет номера телефона.</string>
    <string name="saved">Сохранено</string>
    <string name="deleted">Удалено</string>
</resources>

```

## `app/src/main/res/values/styles.xml`
```xml
<resources>
    <style name="Theme.BuvnieksR" parent="Theme.Material3.DayNight.NoActionBar">
        <item name="colorPrimary">@color/buvnieks_primary</item>
        <item name="colorSecondary">@color/buvnieks_secondary</item>
        <item name="android:fontFamily">sans</item>
        <item name="android:windowLightStatusBar">false</item>
    </style>
</resources>

```

## `app/src/main/res/xml/file_paths.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <files-path name="shared_files" path="shared/" />
</paths>

```

## `build.gradle`
```groovy
plugins {
    id 'com.android.application' version '8.2.2' apply false
    id 'org.jetbrains.kotlin.android' version '1.9.22' apply false
}

```

## `gradle.properties`
```properties
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official

```

## `settings.gradle`
```groovy
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

rootProject.name = "BuvnieksR"
include(":app")

```
