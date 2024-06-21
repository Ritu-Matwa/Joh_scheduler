# Joh_scheduler
#include <iostream>
#include <vector>
#include <string>
#include <thread>
#include <chrono>
#include <algorithm>
#include <unordered_map>
#include <set>
#include <mutex>
#include <condition_variable>
#include <ctime>
using namespace std;

struct Job {
    string name;
    int execution_time;
    vector<string> required_resources;
    vector<string> dependencies;
    int importance;

    Job(string n, int et, vector<string> rr, vector<string> dep, int imp)
        : name(n), execution_time(et), required_resources(rr), dependencies(dep), importance(imp) {}
};

class ResourceManager {
    unordered_map<string, int> resources;
    mutex mtx;
    condition_variable cv;

public:
    ResourceManager(unordered_map<string, int> res) : resources(res) {}

    void acquire_resources(const vector<string>& required_resources) {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [&]() {
            for (const auto& res : required_resources) {
                if (resources[res] <= 0) return false;
            }
            return true;
        });

        for (const auto& res : required_resources) {
            resources[res]--;
        }
    }

    void release_resources(const vector<string>& required_resources) {
        unique_lock<mutex> lock(mtx);
        for (const auto& res : required_resources) {
            resources[res]++;
        }
        cv.notify_all();
    }
};

void execute_job(const Job& job, ResourceManager& resource_manager, unordered_map<string, bool>& finished_jobs, mutex& finished_jobs_mtx, chrono::steady_clock::time_point start_time) {
    resource_manager.acquire_resources(job.required_resources);

    auto job_start_time = chrono::steady_clock::now();
    auto start_seconds = chrono::duration_cast<chrono::seconds>(job_start_time - start_time).count();

    cout << "Job " << job.name << " starting at " << start_seconds << " seconds, requiring resources: ";
    for (const auto& resource : job.required_resources) {
        cout << resource << " ";
    }
    cout << "for " << job.execution_time << " seconds." << endl;

    this_thread::sleep_for(chrono::seconds(job.execution_time));
    cout << "Job " << job.name << " completed." << endl;

    resource_manager.release_resources(job.required_resources);

    lock_guard<mutex> lock(finished_jobs_mtx);
    finished_jobs[job.name] = true;
}

bool dependencies_finished(const Job& job, const unordered_map<string, bool>& finished_jobs) {
    for (const auto& dep : job.dependencies) {
        if (finished_jobs.find(dep) == finished_jobs.end() || !finished_jobs.at(dep)) {
            return false;
        }
    }
    return true;
}

void job_scheduler(vector<Job> jobs, unordered_map<string, int> initial_resources) {
    sort(jobs.begin(), jobs.end(), [](const Job& a, const Job& b) {
        return a.importance > b.importance;
    });

    ResourceManager resource_manager(initial_resources);

    unordered_map<string, bool> finished_jobs;
    mutex finished_jobs_mtx;

    vector<thread> threads;

    auto start_time = chrono::steady_clock::now();

    while (!jobs.empty()) {
        for (auto it = jobs.begin(); it != jobs.end();) {
            if (dependencies_finished(*it, finished_jobs)) {
                threads.emplace_back(execute_job, ref(*it), ref(resource_manager), ref(finished_jobs), ref(finished_jobs_mtx), start_time);
                it = jobs.erase(it);
            } else {
                ++it;
            }
        }

        for (auto it = threads.begin(); it != threads.end();) {
            if (it->joinable()) {
                it->join();
                it = threads.erase(it);
            } else {
                ++it;
            }
        }

        this_thread::sleep_for(chrono::milliseconds(100));
    }

    for (auto& thread : threads) {
        if (thread.joinable()) {
            thread.join();
        }
    }
}

int main() {
    vector<Job> jobs = {
        Job("Job1", 5, {"CPU"}, {}, 1),
        Job("Job2", 3, {"GPU"}, {"Job1"}, 2),
        Job("Job3", 2, {"CPU"}, {"Job2"}, 3),
        Job("Job4", 4, {"GPU"}, {}, 1)
    };

    unordered_map<string, int> initial_resources = {
        {"CPU", 1},
        {"GPU", 1}
    };

    job_scheduler(jobs, initial_resources);

    return 0;
}
