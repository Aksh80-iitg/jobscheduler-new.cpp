#include <iostream>
#include <queue>
#include <vector>
#include <random>
#include <fstream>
#include <algorithm>
#include <limits.h>

using namespace std;

struct Job {
    int jobId;
    int arrivalTime, coresRequired, memoryRequired, executionTime;

    Job(int id, int at, int cr, int mr, int et)
        : jobId(id), arrivalTime(at), coresRequired(cr), memoryRequired(mr), executionTime(et) {}

    int grossValue() const {
        return coresRequired * memoryRequired * executionTime;
    }
};

struct WorkerNode {
    int id, availableCores = 24, availableMemory = 64;
    int totalCores = 24, totalMemory = 64;

    WorkerNode(int id) : id(id) {}

    bool allocate(Job& job) {
        if (availableCores >= job.coresRequired && availableMemory >= job.memoryRequired) {
            availableCores -= job.coresRequired;
            availableMemory -= job.memoryRequired;
            return true;
        }
        return false;
    }

    void release(Job& job) {
        availableCores += job.coresRequired;
        availableMemory += job.memoryRequired;
    }

    double cpuUtilization() const {
        return (1.0 - double(availableCores) / totalCores) * 100;
    }

    double memoryUtilization() const {
        return (1.0 - double(availableMemory) / totalMemory) * 100;
    }
};

class MasterScheduler {
private:
    vector<WorkerNode> workerNodes;
    queue<Job> jobQueue;
    const int maxRetries = 5;
    int currentTime = 0;

    void sortJobQueue(vector<Job>& jobs, const string& queuePolicy) {
        if (queuePolicy == "FCFS") {
            sort(jobs.begin(), jobs.end(), [](const Job& a, const Job& b) {
                return a.arrivalTime < b.arrivalTime;
            });
        } else if (queuePolicy == "smallest") {
            sort(jobs.begin(), jobs.end(), [](const Job& a, const Job& b) {
                return a.grossValue() < b.grossValue();
            });
        } else if (queuePolicy == "duration") {
            sort(jobs.begin(), jobs.end(), [](const Job& a, const Job& b) {
                return a.executionTime < b.executionTime;
            });
        }
    }

    WorkerNode* findFit(Job& job, const string& fitPolicy) {
        WorkerNode* selectedWorker = nullptr;
        if (fitPolicy == "first") {
            for (auto& worker : workerNodes) {
                if (worker.allocate(job)) return &worker;
            }
        } else if (fitPolicy == "best") {
            int minWaste = INT_MAX;
            for (auto& worker : workerNodes) {
                if (worker.allocate(job)) {
                    int waste = (worker.availableCores - job.coresRequired) + (worker.availableMemory - job.memoryRequired);
                    if (waste < minWaste) {
                        minWaste = waste;
                        selectedWorker = &worker;
                    }
                    worker.release(job);
                }
            }
        } else if (fitPolicy == "worst") {
            int maxWaste = 0;
            for (auto& worker : workerNodes) {
                if (worker.allocate(job)) {
                    int waste = (worker.availableCores - job.coresRequired) + (worker.availableMemory - job.memoryRequired);
                    if (waste > maxWaste) {
                        maxWaste = waste;
                        selectedWorker = &worker;
                    }
                    worker.release(job);
                }
            }
        }
        return selectedWorker;
    }

public:
    MasterScheduler(int numWorkers) {
        for (int i = 0; i < numWorkers; i++)
            workerNodes.emplace_back(i);
    }

    void addJob(Job job) {
        jobQueue.push(job);
    }

    void scheduleJobs(string queuePolicy, string fitPolicy, ofstream& outfile) {
        vector<Job> jobs;
        while (!jobQueue.empty()) {
            jobs.push_back(jobQueue.front());
            jobQueue.pop();
        }

        sortJobQueue(jobs, queuePolicy);

        while (!jobs.empty()) {
            currentTime++;
            vector<Job> pendingJobs;
            vector<Job> scheduledJobs;

            for (auto& job : jobs) {
                bool scheduled = false;
                for (int retries = 0; retries < maxRetries && !scheduled; retries++) {
                    WorkerNode* selectedWorker = findFit(job, fitPolicy);
                    if (selectedWorker) {
                        scheduledJobs.push_back(job);
                        scheduled = true;
                        break;
                    }
                }
                if (!scheduled) {
                    pendingJobs.push_back(job);
                }
            }

            sort(scheduledJobs.begin(), scheduledJobs.end(), [](const Job& a, const Job& b) {
                return a.jobId < b.jobId;
            });

            for (const auto& job : scheduledJobs) {
                cout << "JobId: " << job.jobId << " Arrival Day: " << job.arrivalTime / 24 << " Time Hour: " << job.arrivalTime % 24
                     << " MemReq: " << job.memoryRequired << " CPUReq: " << job.coresRequired << " ExeTime: " << job.executionTime << endl;
            }

            jobs = pendingJobs;
            collectAndSaveUtilization(outfile);
        }
    }

    void collectAndSaveUtilization(ofstream& outfile) {
        double totalCpuUtil = 0.0, totalMemUtil = 0.0;
        for (const auto& worker : workerNodes) {
            totalCpuUtil += worker.cpuUtilization();
            totalMemUtil += worker.memoryUtilization();
        }
        outfile << currentTime << "," << totalCpuUtil / workerNodes.size() << "," << totalMemUtil / workerNodes.size() << endl;
    }
};

void generateJobs(MasterScheduler& scheduler, int numJobs) {
    random_device rd;
    mt19937 gen(rd());
    uniform_int_distribution<> arrival(0, 23), cores(1, 24), memory(1, 64), execution(1, 10);

    for (int i = 0; i < numJobs; i++) {
        int arrivalTime = i % (30 * 24); // Spread arrival times across 30 days
        int coresRequired = cores(gen);
        int memoryRequired = memory(gen);
        int executionTime = execution(gen);

        // Ensure memory and CPU requirements are within the given dataset range
        if (memoryRequired > 20) memoryRequired = 20;
        if (coresRequired > 10) coresRequired = 10;

        scheduler.addJob(Job(i + 1, arrivalTime, coresRequired, memoryRequired, executionTime));
    }
}

void generateGraphs() {
    const char* gnuplotScript =
        "set terminal png size 800,600\n"
        "set output 'cpu_utilization.png'\n"
        "set title 'CPU Utilization per Hour'\n"
        "set xlabel 'Hour'\n"
        "set ylabel 'CPU Utilization (%)'\n"
        "set style data histogram\n"
        "set style histogram cluster gap 1\n"
        "set style fill solid 1.00 border -1\n"
        "set boxwidth 0.9\n"
        "plot 're_utilization.csv' using 2:xtic(1) title 'CPU Utilization'\n"
        "set output 'memory_utilization.png'\n"
        "set title 'Memory Utilization per Hour'\n"
        "plot 're_utilization.csv' using 3:xtic(1) title 'Memory re_Utilization'\n";

    std::ofstream scriptFile("gnuplot_script.gp");
    scriptFile << gnuplotScript;
    scriptFile.close();

    std::system("gnuplot gnuplot_script.gp");
}

int main() {
    MasterScheduler scheduler(128);
    generateJobs(scheduler, 5000); // Generate jobs up to the given JobId

    // Prepare output file for CSV
    ofstream outfile("re_utilization.csv");
    outfile << "JobId,CPU Utilization,Memory Utilization\n";

    // Test with different policies
    scheduler.scheduleJobs("FCFS", "first", outfile);
    scheduler.scheduleJobs("smallest", "best", outfile);
    scheduler.scheduleJobs("duration", "worst", outfile);

    // Close the output file
    outfile.close();

    // Generate graphs
    generateGraphs();

    return 0;
}
