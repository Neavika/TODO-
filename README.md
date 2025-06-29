name: todo_app
description: A simple Todo Task Management app for hackathon

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  google_sign_in: ^6.1.4
  provider: ^6.1.1
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  firebase_core: ^3.0.0
  firebase_crashlytics: ^4.0.0
  flutter_slidable: ^3.0.0
  intl: ^0.19.0

dart
Copy
Edit
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'package:provider/provider.dart';
import 'screens/login_screen.dart';
import 'providers/task_provider.dart';
import 'services/crashlytics_service.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  await Hive.initFlutter();
  await Hive.openBox('tasks');
  CrashlyticsService.initialize();

  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (_) => TaskProvider(),
      child: MaterialApp(
        title: 'Todo App',
        theme: ThemeData(primarySwatch: Colors.indigo),
        home: LoginScreen(),
        debugShowCheckedModeBanner: false,
      ),
    );
  }
}

dart
Copy
Edit
import 'package:google_sign_in/google_sign_in.dart';

class AuthService {
  final GoogleSignIn _googleSignIn = GoogleSignIn();

  Future<GoogleSignInAccount?> signInWithGoogle() async {
    try {
      return await _googleSignIn.signIn();
    } catch (e) {
      print("Login Error: $e");
      return null;
    }
  }

  Future<void> signOut() async {
    await _googleSignIn.signOut();
  }
}

dart
Copy
Edit
import 'package:firebase_crashlytics/firebase_crashlytics.dart';

class CrashlyticsService {
  static void initialize() {
    FirebaseCrashlytics.instance.setCrashlyticsCollectionEnabled(true);
  }

  static void log(String message) {
    FirebaseCrashlytics.instance.log(message);
  }
}

dart
Copy
Edit
class Task {
  String title;
  String description;
  DateTime dueDate;
  bool isComplete;

  Task({
    required this.title,
    required this.description,
    required this.dueDate,
    this.isComplete = false,
  });
}

dart
Copy
Edit
import 'package:flutter/material.dart';
import '../models/task_model.dart';

class TaskProvider extends ChangeNotifier {
  List<Task> _tasks = [];

  List<Task> get tasks => _tasks;

  void addTask(Task task) {
    _tasks.add(task);
    notifyListeners();
  }

  void updateTask(int index, Task task) {
    _tasks[index] = task;
    notifyListeners();
  }

  void deleteTask(int index) {
    _tasks.removeAt(index);
    notifyListeners();
  }

  void toggleComplete(int index) {
    _tasks[index].isComplete = !_tasks[index].isComplete;
    notifyListeners();
  }
}

dart
Copy
Edit
import 'package:flutter/material.dart';
import '../services/auth_service.dart';
import 'home_screen.dart';

class LoginScreen extends StatelessWidget {
  final AuthService _authService = AuthService();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: ElevatedButton.icon(
          icon: Icon(Icons.login),
          label: Text("Sign in with Google"),
          onPressed: () async {
            var user = await _authService.signInWithGoogle();
            if (user != null) {
              Navigator.pushReplacement(
                context,
                MaterialPageRoute(builder: (_) => HomeScreen()),
              );
            } else {
              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(content: Text("Login Failed")),
              );
            }
          },
        ),
      ),
    );
  }
}

dart
Copy
Edit
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../models/task_model.dart';
import '../providers/task_provider.dart';
import 'task_form_screen.dart';
import '../services/auth_service.dart';

class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    var tasks = context.watch<TaskProvider>().tasks;

    return Scaffold(
      appBar: AppBar(
        title: Text('My Tasks'),
        actions: [
          IconButton(
            icon: Icon(Icons.logout),
            onPressed: () {
              AuthService().signOut();
              Navigator.pop(context);
            },
          )
        ],
      ),
      body: tasks.isEmpty
          ? Center(child: Text('No tasks yet'))
          : ListView.builder(
              itemCount: tasks.length,
              itemBuilder: (ctx, index) {
                final task = tasks[index];
                return Dismissible(
                  key: ValueKey(task.title + index.toString()),
                  onDismissed: (_) =>
                      context.read<TaskProvider>().deleteTask(index),
                  child: ListTile(
                    title: Text(
                      task.title,
                      style: TextStyle(
                        decoration:
                            task.isComplete ? TextDecoration.lineThrough : null,
                      ),
                    ),
                    subtitle: Text(task.description),
                    trailing: IconButton(
                      icon: Icon(
                          task.isComplete
                              ? Icons.check_circle
                              : Icons.radio_button_unchecked,
                          color: task.isComplete ? Colors.green : Colors.grey),
                      onPressed: () =>
                          context.read<TaskProvider>().toggleComplete(index),
                    ),
                    onTap: () {
                      Navigator.push(
                        context,
                        MaterialPageRoute(
                          builder: (_) => TaskFormScreen(
                            index: index,
                            existingTask: task,
                          ),
                        ),
                      );
                    },
                  ),
                );
              },
            ),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () => Navigator.push(
          context,
          MaterialPageRoute(builder: (_) => TaskFormScreen()),
        ),
      ),
    );
  }
}

dart
Copy
Edit
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../models/task_model.dart';
import '../providers/task_provider.dart';

class TaskFormScreen extends StatefulWidget {
  final int? index;
  final Task? existingTask;

  TaskFormScreen({this.index, this.existingTask});

  @override
  _TaskFormScreenState createState() => _TaskFormScreenState();
}

class _TaskFormScreenState extends State<TaskFormScreen> {
  final _formKey = GlobalKey<FormState>();
  late String _title;
  late String _description;
  DateTime _dueDate = DateTime.now();

  @override
  void initState() {
    super.initState();
    if (widget.existingTask != null) {
      _title = widget.existingTask!.title;
      _description = widget.existingTask!.description;
      _dueDate = widget.existingTask!.dueDate;
    } else {
      _title = '';
      _description = '';
    }
  }

  @override
  Widget build(BuildContext context) {
    final isEdit = widget.existingTask != null;

    return Scaffold(
      appBar: AppBar(title: Text(isEdit ? 'Edit Task' : 'New Task')),
      body: Padding(
        padding: EdgeInsets.all(16),
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              TextFormField(
                initialValue: _title,
                decoration: InputDecoration(labelText: 'Title'),
                onSaved: (val) => _title = val!,
              ),
              TextFormField(
                initialValue: _description,
                decoration: InputDecoration(labelText: 'Description'),
                onSaved: (val) => _description = val!,
              ),
              SizedBox(height: 16),
              ElevatedButton(
                onPressed: () => _selectDate(context),
                child: Text("Due: ${_dueDate.toLocal()}".split(' ')[0]),
              ),
              Spacer(),
              ElevatedButton(
                onPressed: () {
                  _formKey.currentState!.save();
                  final task = Task(
                    title: _title,
                    description: _description,
                    dueDate: _dueDate,
                  );

                  if (isEdit) {
                    context.read<TaskProvider>().updateTask(widget.index!, task);
                  } else {
                    context.read<TaskProvider>().addTask(task);
                  }

                  Navigator.pop(context);
                },
                child: Text(isEdit ? 'Update Task' : 'Add Task'),
              )
            ],
          ),
        ),
      ),
    );
  }

  Future<void> _selectDate(BuildContext context) async {
    final picked = await showDatePicker(
        context: context,
        initialDate: _dueDate,
        firstDate: DateTime(2023),
        lastDate: DateTime(2100));
    if (picked != null) setState(() => _dueDate = picked);
  }
}
