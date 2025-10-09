# Lab 01 - Start the Web Server and Test the Application

## Objective

Get the ResumeCrunch application up and running using Docker Compose, then test its core functionality by uploading resumes and analyzing job description matches.

## Background

The ResumeCrunch application is a web-based resume parsing and scoring service that uses:
- **Flask** - Python web framework
- **Nginx** - Web server and reverse proxy
- **Redis** - Caching layer
- **Docker** - Containerization

The application analyzes resumes against job descriptions and provides scoring to help with candidate evaluation.

## Steps

### 1. Clone the Repository (if not already done)

```bash
git clone https://github.com/yourusername/ResumeCrunch.git
cd ResumeCrunch
```

### 2. Start the Application

Use Docker Compose to start all services:

```bash
docker-compose up --build -d
```

This command will:
- Build the application containers
- Start Nginx, Flask app, and Redis
- Run everything in the background (`-d` flag)

### 3. Verify Services are Running

Check that all containers are up:

```bash
docker-compose ps
```

You should see three containers running:
- `resumecrunch-nginx`
- `resumecrunch-app`
- `resumecrunch-redis`

### 4. Access the Application

Open your browser and navigate to:

```
http://localhost:8080
```

You should see the ResumeCrunch home page.

### 5. Test Application Functionality

1. Click on Resumes on the left side
2. Click on the file upload button area
3. Select `my_resume.zip` file from the lab-files project directory
   - This zip file contains fake PDF resume sample in the same directory (my_resume.pdf)
4. Click "Upload" to submit the zip file

The application will:
- Extract the zip file
- Store them for later analysis

### 6. Test the Resume Scoring Feature

Now test the core functionality by scoring resumes against a job description:

1. Navigate to ResumeCrunch page

2. Select a resume to analyze: *my_resume.pdf*

3. Enter a job description
   - A job description is provided in the lab-files directory as `job_description.txt`

   Example job description:
   ```
   As a Machine Learning Engineer in OpenAI's Applied Group, you will have the opportunity to work with some of the brightest minds in AI. You'll contribute to deploying state-of-the-art models in production environments, helping turn research breakthroughs into tangible solutions that improve the trust and safety of our platform. If you're excited about fine tuning LLMs and building ML models this role is your chance to make a significant mark.
    In this role, you will:
    Innovate and Deploy: Design and deploy advanced machine learning models that solve real-world problems. Bring OpenAI's research from concept to implementation, creating AI-driven applications with a direct impact.
    Collaborate with the Best: Work closely with researchers, software engineers, and product managers to understand complex business challenges and deliver AI-powered solutions. Be part of a dynamic team where ideas flow freely and creativity thrives.
    Optimize and Scale: Implement scalable data pipelines, optimize models for performance and accuracy, and ensure they are production-ready. Contribute to projects that require cutting-edge technology and innovative approaches.
    Learn and Lead: Stay ahead of the curve by engaging with the latest developments in machine learning and AI. Take part in code reviews, share knowledge, and lead by example to maintain high-quality engineering practices.
    Make a Difference: Monitor and maintain deployed models to ensure they continue delivering value. Your work will directly influence how AI benefits individuals, businesses, and society at large.
    You might thrive in this role if you:
    Master's/  PhD degree in Computer Science, Machine Learning, Data Science, or a related field. 
    Demonstrated experience in deep learning and transformers models
    Proficiency in frameworks like PyTorch or Tensorflow
    Strong foundation in data structures, algorithms, and software engineering principles.
    Experience with search relevance, ads ranking  or LLMs is a plus.
    Are familiar with methods of training and fine-tuning large language models, such as distillation, supervised fine-tuning, and policy optimization
    Excellent problem-solving and analytical skills, with a proactive approach to challenges.
    Ability to work collaboratively with cross-functional teams.
    Ability to move fast in an environment where things are sometimes loosely defined and may have competing priorities or deadlines
    Enjoy owning the problems end-to-end, and are willing to pick up whatever knowledge you're missing to get the job done
   ```
4. Click Evaluate Resume
    - The application will process the resume and job description
    - You may see a loading indicator while the analysis is performed

**Note:** If you do not have an OpenAI API key set, the application may not be able to perform the analysis. You can set the `OPENAI_API_KEY` environment variable in your shell before starting the Docker containers if you want to test with real API calls. Using your own key, of course.

## Verification

- [ ] All Docker containers are running
- [ ] Application is accessible at http://localhost:8080
- [ ] You can view the home page
- [ ] Successfully uploaded a zip file with resume or resumes
- [ ] Application extracted and processed the resume batch
- [ ] Entered a job description and received scoring results
- [ ] Viewed scores for resume + job description combinations
- [ ] Explored the application's features and interface

## What You've Learned

At this point, you should understand:
- How the ResumeCrunch application works
- The resume upload process (via zip files)
- How job descriptions are matched against resumes
- The basic user workflow and features

## Next Steps

Now that you understand how the application works, it's time to begin security testing. Proceed to [Lab 02 - Interrogate the Web Server](lab-02-interrogate-server.md) to start reconnaissance and discover potential vulnerabilities.

---

[← Back to Lab Overview](lab-overview.md) | [Next: Interrogate Server →](lab-02-interrogate-server.md)
