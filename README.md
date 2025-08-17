#include <iostream>
#include <vector>
#include <string>
#include <fstream>
#include <algorithm>
#include <ctime>
#include <sstream>

using namespace std;

class Event
{
private:
    int id;
    string name;
    string date; // DD-MM-YYYY
    string time; // HH:MM
    string type;
    string location;
    vector<string> attendees; // emails

public:
    Event() : id(0), name(""), date(""), time(""), type(""), location("") {}
    Event(int i, string n, string d, string t, string ty, string l)
        : id(i), name(n), date(d), time(t), type(ty), location(l) {}

    int getID() const { return id; }
    string getName() const { return name; }
    string getDate() const { return date; }
    string getTime() const { return time; }
    string getType() const { return type; }
    string getLocation() const { return location; }
    // non-const and const overloads (fixes the 4 errors)
    vector<string> &getAttendees() { return attendees; }
    const vector<string> &getAttendees() const { return attendees; }

    void setName(const string &n) { name = n; }
    void setDate(const string &d) { date = d; }
    void setTime(const string &t) { time = t; }
    void setType(const string &ty) { type = ty; }
    void setLocation(const string &l) { location = l; }
    void setAttendees(const vector<string> &att) { attendees = att; }

    bool operator==(const Event &other) const
    {
        // Treat same name+date+time+type as duplicate (by design)
        return name == other.name && date == other.date && time == other.time && type == other.type;
    }

    void display() const
    {
        cout << "ID: " << id << " | Name: " << name << " | Date: " << date << " | Time: " << time
             << " | Type: " << type << " | Location: " << location << endl;
        cout << "Attendees: ";
        if (attendees.empty())
            cout << "None";
        else
        {
            for (const auto &email : attendees)
                cout << email << " ";
        }
        cout << endl;
    }
};

class EventManager
{
private:
    vector<Event> events;
    int nextID;

    // -------- File I/O helpers --------
    void loadEvents()
    {
        ifstream fin("events.db");
        if (!fin) {
            nextID = 1;
            return;
        }

        events.clear();
        nextID = 1;

        string line;
        while (getline(fin, line))
        {
            if (line.empty()) continue;

            int id = 0;
            try { id = stoi(line); }
            catch (...) { break; }

            string data;
            if (!getline(fin, data)) break; // second line must exist

            stringstream ss(data);
            string name, date, time, type, location, attendeeStr;
            getline(ss, name, '|');
            getline(ss, date, '|');
            getline(ss, time, '|');
            getline(ss, type, '|');
            getline(ss, location, '|');
            getline(ss, attendeeStr);

            Event ev(id, name, date, time, type, location);

            if (!attendeeStr.empty())
            {
                vector<string> attEmails;
                stringstream ss2(attendeeStr);
                string email;
                while (getline(ss2, email, ','))
                {
                    if (!email.empty())
                        attEmails.push_back(email);
                }
                ev.setAttendees(attEmails);
            }
            events.push_back(ev);
            if (id >= nextID) nextID = id + 1;
        }
        fin.close();
    }

    void saveEvents()
    {
        ofstream fout("events.db");
        for (const auto &e : events)
        {
            fout << e.getID() << "\n"
                 << e.getName() << "|"
                 << e.getDate() << "|" << e.getTime() << "|" << e.getType() << "|"
                 << e.getLocation() << "|";

            const vector<string> &att = e.getAttendees();
            for (size_t i = 0; i < att.size(); ++i)
            {
                fout << att[i];
                if (i + 1 < att.size())
                    fout << ",";
            }
            fout << "\n";
        }
        fout.close();
    }

    // -------- Validation helpers --------
    bool validateDate(const string &date)
    {
        if (date.size() != 10 || date[2] != '-' || date[5] != '-') return false;
        try
        {
            int day = stoi(date.substr(0, 2));
            int month = stoi(date.substr(3, 2));
            int year = stoi(date.substr(6, 4));
            if (year < 1900 || month < 1 || month > 12 || day < 1) return false;

            static const int mdays[13] = {0,31,28,31,30,31,30,31,31,30,31,30,31};
            int maxd = mdays[month];
            bool leap = (year%4==0 && (year%100!=0 || year%400==0));
            if (month == 2 && leap) maxd = 29;
            return day <= maxd;
        }
        catch (...) { return false; }
    }

    bool validateTime(const string &t)
    {
        if (t.size() != 5 || t[2] != ':') return false;
        try
        {
            int hr = stoi(t.substr(0, 2));
            int min = stoi(t.substr(3, 2));
            return (0 <= hr && hr <= 23 && 0 <= min && min <= 59);
        }
        catch (...) { return false; }
    }

    bool isDuplicate(const Event &newEvent)
    {
        return any_of(events.begin(), events.end(),
                      [&](const Event &e) { return e == newEvent; });
    }

public:
    EventManager() : nextID(1) { loadEvents(); }
    ~EventManager() { saveEvents(); }

    // -------- Features --------
    void addEvent()
    {
        string name, date, time, type, location;
        cout << "Enter Event Name: ";
        getline(cin, name);

        cout << "Enter Date (DD-MM-YYYY): ";
        getline(cin, date);
        if (!validateDate(date))
        {
            cout << "Invalid date format.\n";
            return;
        }

        cout << "Enter Time (HH:MM): ";
        getline(cin, time);
        if (!validateTime(time))
        {
            cout << "Invalid time format.\n";
            return;
        }

        cout << "Enter Type: ";
        getline(cin, type);

        cout << "Enter Location (optional): ";
        getline(cin, location);

        Event newEvent(nextID, name, date, time, type, location);

        // Attendees from file (optional)
        cout << "Enter attendee email list file (e.g. attendees.csv), or leave blank to skip: ";
        string filename;
        getline(cin, filename);
        vector<string> attendees;
        if (!filename.empty())
        {
            ifstream fin(filename);
            if (!fin)
            {
                cout << "Could not open attendees file.\n";
            }
            else
            {
                string email;
                while (getline(fin, email))
                {
                    if (!email.empty())
                        attendees.push_back(email);
                }
                fin.close();
                cout << "Loaded " << attendees.size() << " attendees.\n";
            }
        }
        newEvent.setAttendees(attendees);

        if (isDuplicate(newEvent))
        {
            cout << "Duplicate event exists.\n";
            return;
        }

        events.push_back(newEvent);
        ++nextID;
        saveEvents();
        cout << "Event added successfully.\n";
    }

    void sendReminders(int daysAhead = 1)
    {
        time_t now = time(nullptr);
        bool found = false;

        auto parseDate = [](const string &dateStr) -> tm {
            tm dateTm = {};
            dateTm.tm_mday = stoi(dateStr.substr(0, 2));
            dateTm.tm_mon  = stoi(dateStr.substr(3, 2)) - 1;
            dateTm.tm_year = stoi(dateStr.substr(6, 4)) - 1900;
            dateTm.tm_hour = 0;
            dateTm.tm_min  = 0;
            dateTm.tm_sec  = 0;
            return dateTm;
        };

        // Normalize "today" to midnight for day-diff comparisons
        tm *lt = localtime(&now);
        tm todayMid = *lt;
        todayMid.tm_hour = todayMid.tm_min = todayMid.tm_sec = 0;
        time_t today = mktime(&todayMid);

        for (const auto &e : events)
        {
            if (e.getAttendees().empty()) continue;

            tm eventDateTm = parseDate(e.getDate());
            time_t eventDay = mktime(&eventDateTm);

            double diffDays = difftime(eventDay, today) / (60 * 60 * 24);

            if (diffDays >= 0 && diffDays <= daysAhead)
            {
                found = true;
                for (const string &email : e.getAttendees())
                {
                    cout << "Sending reminder to " << email << " for event "
                         << e.getName() << " on " << e.getDate() << " at " << e.getTime() << endl;

                    // Demo command (requires 'mail' setup on Unix). Left commented by default.
                    // stringstream cmd;
                    // cmd << "echo \"Reminder: Event '" << e.getName() << "' scheduled on "
                    //     << e.getDate() << " at " << e.getTime()
                    //     << "\" | mail -s \"Event Reminder\" " << email;
                    // system(cmd.str().c_str());
                }
            }
        }
        if (!found)
            cout << "No upcoming events with attendees for reminder.\n";
    }

    void searchEvent()
    {
        cout << "Enter search keyword (name, type, or location): ";
        string keyword;
        getline(cin, keyword);

        bool found = false;
        for (const auto &e : events)
        {
            if (e.getName().find(keyword) != string::npos ||
                e.getType().find(keyword) != string::npos ||
                e.getLocation().find(keyword) != string::npos)
            {
                e.display();
                found = true;
            }
        }
        if (!found)
            cout << "No events found matching the keyword.\n";
    }

    void viewEventsByDate(const string &date)
    {
        if (!validateDate(date))
        {
            cout << "Invalid date format.\n";
            return;
        }
        bool found = false;
        for (const auto &e : events)
        {
            if (e.getDate() == date)
            {
                e.display();
                found = true;
            }
        }
        if (!found)
            cout << "No events found on " << date << ".\n";
    }

    void viewTodaysEvents()
    {
        time_t now = time(nullptr);
        tm *ltm = localtime(&now);
        char buf[11];
        strftime(buf, sizeof(buf), "%d-%m-%Y", ltm);
        string todayStr(buf);
        viewEventsByDate(todayStr);
    }

    void showAllEvents()
    {
        if (events.empty())
        {
            cout << "No events available.\n";
            return;
        }
        for (const auto &e : events)
            e.display();
    }

    void editEvent()
    {
        cout << "Enter event ID to edit: ";
        string idStr;
        getline(cin, idStr);
        int id;
        try { id = stoi(idStr); }
        catch (...) { cout << "Invalid ID format.\n"; return; }

        auto it = find_if(events.begin(), events.end(), [id](const Event &e) { return e.getID() == id; });
        if (it == events.end())
        {
            cout << "Event not found.\n";
            return;
        }
        Event &e = *it;

        cout << "Editing Event ID " << id << ":\n";

        cout << "Current Name: " << e.getName() << ". Enter new name or leave blank to keep: ";
        string input;
        getline(cin, input);
        if (!input.empty()) e.setName(input);

        cout << "Current Date: " << e.getDate() << ". Enter new date (DD-MM-YYYY) or leave blank: ";
        getline(cin, input);
        if (!input.empty())
        {
            if (!validateDate(input))
            {
                cout << "Invalid date format. Edit cancelled.\n";
                return;
            }
            e.setDate(input);
        }

        cout << "Current Time: " << e.getTime() << ". Enter new time (HH:MM) or leave blank: ";
        getline(cin, input);
        if (!input.empty())
        {
            if (!validateTime(input))
            {
                cout << "Invalid time format. Edit cancelled.\n";
                return;
            }
            e.setTime(input);
        }

        cout << "Current Type: " << e.getType() << ". Enter new type or leave blank: ";
        getline(cin, input);
        if (!input.empty()) e.setType(input);

        cout << "Current Location: " << e.getLocation() << ". Enter new location or leave blank: ";
        getline(cin, input);
        if (!input.empty()) e.setLocation(input);

        saveEvents();
        cout << "Event updated successfully.\n";
    }

    void deleteEvent()
    {
        cout << "Enter event ID to delete: ";
        string idStr;
        getline(cin, idStr);
        int id;
        try { id = stoi(idStr); }
        catch (...) { cout << "Invalid ID format.\n"; return; }

        auto it = remove_if(events.begin(), events.end(), [id](const Event &e) { return e.getID() == id; });
        if (it != events.end())
        {
            events.erase(it, events.end());
            saveEvents();
            cout << "Event deleted successfully.\n";
        }
        else
        {
            cout << "Event not found.\n";
        }
    }
};

void adminLogin(bool &isAdmin)
{
    cout << "Enter admin password: ";
    string pass;
    getline(cin, pass);
    if (pass == "admin123")
    {
        isAdmin = true;
        cout << "Admin login successful.\n";
    }
    else
    {
        cout << "Incorrect password.\n";
    }
}

int main()
{
    EventManager manager;
    string choice;
    bool isAdmin = false;

    while (true)
    {
        cout << "\n==== Smart Event Manager ====\n";
        if (!isAdmin)
        {
            cout << "1. Search Event\n2. View Events by Date\n3. View Today's Events\n4. View All Events\n5. Admin Login\n6. Exit\nChoice: ";
            getline(cin, choice);
            if (choice == "1")
                manager.searchEvent();
            else if (choice == "2")
            {
                cout << "Enter Date (DD-MM-YYYY): ";
                string date;
                getline(cin, date);
                manager.viewEventsByDate(date);
            }
            else if (choice == "3")
                manager.viewTodaysEvents();
            else if (choice == "4")
                manager.showAllEvents();
            else if (choice == "5")
                adminLogin(isAdmin);
            else if (choice == "6")
                break;
            else
                cout << "Invalid choice.\n";
        }
        else
        {
            cout << "1. Add Event\n2. Edit Event\n3. Delete Event\n4. Search Event\n5. View Events by Date\n6. View Today's Events\n7. View All Events\n8. Send Reminders\n9. Logout Admin\n10. Exit\nChoice: ";
            getline(cin, choice);
            if (choice == "1")
                manager.addEvent();
            else if (choice == "2")
                manager.editEvent();
            else if (choice == "3")
                manager.deleteEvent();
            else if (choice == "4")
                manager.searchEvent();
            else if (choice == "5")
            {
                cout << "Enter Date (DD-MM-YYYY): ";
                string date;
                getline(cin, date);
                manager.viewEventsByDate(date);
            }
            else if (choice == "6")
                manager.viewTodaysEvents();
            else if (choice == "7")
                manager.showAllEvents();
            else if (choice == "8")
            {
                cout << "Send reminders for events occurring within how many days? (enter number): ";
                int days = 1;
                string daysStr;
                getline(cin, daysStr);
                try { days = stoi(daysStr); }
                catch (...) { days = 1; }
                if (days < 0) days = 0;
                manager.sendReminders(days);
            }
            else if (choice == "9")
            {
                isAdmin = false;
                cout << "Logged out from admin.\n";
            }
            else if (choice == "10")
                break;
            else
                cout << "Invalid choice.\n";
        }
    }
    return 0;
}
