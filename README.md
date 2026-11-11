#include <iostream>
#include <fstream>
#include <vector>
#include <string>
#include <algorithm>
#include <ctime>
using namespace std;

class Event {
public:
    int id, seats, registered = 0, ratingSum = 0, ratings = 0;
    string name, date, time, category;
    vector<string> attendees, waiting;

    Event(int i, string n, string d, string t,
          string c, int s)
        : id(i), name(n), date(d), time(t),
          category(c), seats(s) {}

    virtual void show() {
        cout << "\nID: " << id << " | " << name
             << " | " << category << " | " << date
             << " " << time << " | Seats left: "
             << seats - registered << "\n";
    }

    virtual ~Event() {}
};

// Inheritance
class Competition : public Event {
public:
    int points = 0;

    Competition(int i, string n, string d, string t,
                string c, int s)
        : Event(i, n, d, t, c, s) {}

    // Runtime polymorphism
    void show() override {
        Event::show();
        cout << "Type: Competition | Points: "
             << points << "\n";
    }
};

// Base class for users
class User {
protected:
    string username;

public:
    User(string u) : username(u) {}
    virtual void menu() {
        cout << "Welcome, " << username << "\n";
    }
    virtual ~User() {}
};

class Student : public User {
public:
    string interest;

    Student(string u, string i) : User(u), interest(i) {}

    void menu() override {
        cout << "\nStudent: " << username
             << " | Interest: " << interest << "\n";
    }
};

class Organizer : public User {
public:
    Organizer(string u) : User(u) {}

    void menu() override {
        cout << "\nOrganizer: " << username << "\n";
    }
};

class CampusManager {
    vector<Event*> events;

    void save() {
        ofstream f("events.txt");
        for (auto e : events)
            f << e->id << "|" << e->name << "|"
              << e->date << "|" << e->time << "|"
              << e->category << "|" << e->seats << "\n";
    }

    void notify(string msg) {
        ofstream f("notifications.txt", ios::app);
        f << msg << "\n";
        cout << "Notification: " << msg << "\n";
    }

    Event* findEvent(int id) {
        for (auto e : events)
            if (e->id == id) return e;
        return nullptr;
    }

public:
    ~CampusManager() {
        for (auto e : events) delete e;
    }

    void load() {
        ifstream f("events.txt");
        string line;
        while (getline(f, line)) {
            
            // Basic pipe-separated file reading
            vector<string> v;
            size_t start = 0, pos;
            while ((pos = line.find('|', start)) != string::npos) {
                v.push_back(line.substr(start, pos - start));
                start = pos + 1;
            }
            v.push_back(line.substr(start));

            if (v.size() == 6) {
                int id = stoi(v[0]), seats = stoi(v[5]);
                if (!findEvent(id))
                    events.push_back(new Event(
                        id, v[1], v[2], v[3], v[4], seats));
            }
        }
    }

    void createEvent() {
        int id, seats, type;
        string n, d, t, c;
        cout << "ID: "; cin >> id;
        if (findEvent(id)) {
            cout << "ID already exists!\n"; return;
        }
        cin.ignore();
        cout << "Event name: "; getline(cin, n);
        cout << "Date (DD-MM-YYYY): "; cin >> d;
        cout << "Time (e.g. 10AM): "; cin >> t;
        cout << "Category: "; cin >> c;
        cout << "Seats: "; cin >> seats;
        cout << "1. Normal event  2. Competition: ";
        cin >> type;

        if (seats <= 0) {
            cout << "Seats must be positive.\n"; return;
        }

        if (type == 2)
            events.push_back(new Competition(id,n,d,t,c,seats));
        else
            events.push_back(new Event(id,n,d,t,c,seats));

        save();
        notify("New event created: " + n);
    }

    void listEvents() {
        if (events.empty()) cout << "No events available.\n";
        for (auto e : events) e->show();
    }

    void searchEvent() {
        string key;
        cout << "Enter name/category: "; cin >> key;
        bool found = false;
        for (auto e : events)
            if (e->name.find(key) != string::npos ||
                e->category.find(key) != string::npos) {
                e->show(); found = true;
            }
        if (!found) cout << "No matching events.\n";
    }

    void registerStudent(string student) {
        int id;
        cout << "Event ID: "; cin >> id;
        Event* e = findEvent(id);
        if (!e) { cout << "Event not found.\n"; return; }

        // Schedule-conflict detection
        for (auto x : events)
            if (x->date == e->date && x->time == e->time &&
                find(x->attendees.begin(), x->attendees.end(),
                     student) != x->attendees.end()) {
                cout << "Schedule conflict!\n"; return;
            }

        if (find(e->attendees.begin(), e->attendees.end(),
                 student) != e->attendees.end() ||
            find(e->waiting.begin(), e->waiting.end(),
                 student) != e->waiting.end()) {
            cout << "Already registered or waitlisted.\n"; return;
        }

        if (e->registered < e->seats) {
            e->attendees.push_back(student);
            e->registered++;
            cout << "\nDIGITAL EVENT PASS\nStudent: " << student
                 << "\nEvent: " << e->name
                 << "\nPass ID: " << e->id << "-" << student << "\n";
            notify(student + " registered for " + e->name);
        } else {
            e->waiting.push_back(student);
            cout << "Seats full. Added to waiting list.\n";
        }
    }

    void cancelEvent() {
        int id;
        cout << "Event ID to cancel: "; cin >> id;
        auto it = find_if(events.begin(), events.end(),
            [id](Event* e) { return e->id == id; });
        if (it == events.end()) {
            cout << "Event not found.\n"; return;
        }
        notify("Event cancelled: " + (*it)->name);
        delete *it;
        events.erase(it);
        save();
    }

    void attendance() {
        int id; string student;
        cout << "Event ID: "; cin >> id;
        Event* e = findEvent(id);
        if (!e) { cout << "Event not found.\n"; return; }
        cout << "Student username: "; cin >> student;

        if (find(e->attendees.begin(), e->attendees.end(),
                 student) == e->attendees.end()) {
            cout << "Student is not registered.\n"; return;
        }

        ofstream f("attendance.txt", ios::app);
        f << e->name << " | " << student << " | Present\n";
        cout << "Attendance marked.\n";
    }

    void feedback() {
        int id, rating;
        cout << "Event ID: "; cin >> id;
        Event* e = findEvent(id);
        if (!e) { cout << "Event not found.\n"; return; }
        cout << "Rating (1-5): "; cin >> rating;
        if (rating < 1 || rating > 5) {
            cout << "Invalid rating.\n"; return;
        }
        e->ratingSum += rating;
        e->ratings++;
        ofstream f("feedback.txt", ios::app);
        f << e->name << " | " << rating << "/5\n";
        cout << "Average rating: "
             << (double)e->ratingSum / e->ratings << "\n";
    }

    void recommendations(string interest) {
        cout << "Recommended events:\n";
        for (auto e : events)
            if (e->category == interest) e->show();
    }

    void leaderboard() {
        cout << "\nCOMPETITION LEADERBOARD\n";
        for (auto e : events) {
            Competition* c = dynamic_cast<Competition*>(e);
            if (c) {
                cout << c->name << " | Points: " << c->points << "\n";
            }
        }
        cout << "Organizer can award points by editing Competition::points.\n";
    }
};

int main() {
    CampusManager manager;
    manager.load();

    int role;
    cout << "=== CAMPUS EVENT MANAGER ===\n";
    cout << "1. Student Login\n2. Organizer Login\nChoice: ";
    cin >> role;

    string username, interest = "";
    cout << "Username: "; cin >> username;

    if (role == 1) {
        cout << "Your interest (e.g. Coding, Sports): ";
        cin >> interest;
    } else if (role != 2) {
        cout << "Invalid role.\n"; return 0;
    }

    User* user;
    if (role == 1) user = new Student(username, interest);
    else user = new Organizer(username);
    user->menu();

    int choice;
    do {
        cout << "\n1.Create Event  2.List Events  3.Search Events"
             << "\n4.Register  5.Cancel Event  6.Attendance"
             << "\n7.Rate Event  8.Recommendations"
             << "\n9.Leaderboard  0.Exit\nChoice: ";
        cin >> choice;

        switch (choice) {
            case 1:
                if (role == 2) manager.createEvent();
                else cout << "Organizer access required.\n";
                break;
            case 2: manager.listEvents(); break;
            case 3: manager.searchEvent(); break;
            case 4:
                if (role == 1) manager.registerStudent(username);
                else cout << "Student login required.\n";
                break;
            case 5:
                if (role == 2) manager.cancelEvent();
                else cout << "Organizer access required.\n";
                break;
            case 6:
                if (role == 2) manager.attendance();
                else cout << "Organizer access required.\n";
                break;
            case 7:
                if (role == 1) manager.feedback();
                else cout << "Student login required.\n";
                break;
            case 8:
                if (role == 1) manager.recommendations(interest);
                else cout << "Student login required.\n";
                break;
            case 9: manager.leaderboard(); break;
        }
    } while (choice != 0);

    delete user;
    return 0;
}
