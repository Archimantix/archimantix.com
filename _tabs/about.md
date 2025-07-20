---
# the default layout is 'page'
icon: fas fa-info-circle
order: 1
---

<style>
.custom-about-content {
    font-family: 'Source Sans Pro', 'Microsoft Yahei', sans-serif;
    line-height: 1.6;
    color: var(--text-color);
}

.custom-about-content .about-container {
    max-width: 100%;
    margin: 0;
    background: transparent;
    border-radius: 0;
    padding: 0;
    box-shadow: none;
    backdrop-filter: none;
}

.custom-about-content .profile-section {
    display: flex;
    align-items: stretch;
    gap: 40px;
    margin-bottom: 40px;
    flex-wrap: wrap;
}

.custom-about-content .profile-picture {
    width: 250px;
    height: 250px;
    border-radius: 50%;
    object-fit: cover;
    border: 5px solid #fff;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
    flex-shrink: 0;
    display: block;
}

.custom-about-content .profile-info {
    flex: 1;
    min-width: 300px;
    display: flex;
    flex-direction: column;
    justify-content: center;
}

.custom-about-content .name {
    font-size: 2.8em;
    font-weight: 700;
    color: var(--heading-color);
    margin-bottom: 10px;
    background: linear-gradient(45deg, #547159, #3a4e3f);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
}

.custom-about-content .position {
    font-size: 1.4em;
    color: var(--text-muted-color);
    margin-bottom: 25px;
    font-weight: 300;
}

.custom-about-content .links {
    display: flex;
    gap: 20px;
    margin-bottom: 0;
    flex-wrap: wrap;
}

.custom-about-content .link {
    width: 50px;
    height: 50px;
    background: linear-gradient(45deg, #547159, #3a4e3f);
    color: white;
    text-decoration: none;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 20px;
    transition: all 0.3s ease;
    box-shadow: 0 4px 15px rgba(84, 113, 89, 0.3);
    position: relative;
}

.custom-about-content .link:hover {
    transform: translateY(-3px);
    box-shadow: 0 6px 20px rgba(84, 113, 89, 0.4);
}

.custom-about-content .link::after {
    content: attr(data-tooltip);
    position: absolute;
    bottom: -35px;
    left: 50%;
    transform: translateX(-50%);
    background: rgba(0, 0, 0, 0.8);
    color: white;
    padding: 5px 10px;
    border-radius: 4px;
    font-size: 12px;
    white-space: nowrap;
    opacity: 0;
    visibility: hidden;
    transition: all 0.3s ease;
}

.custom-about-content .link:hover::after {
    opacity: 1;
    visibility: visible;
}

.custom-about-content .about-text {
    background: var(--card-bg);
    padding: 30px;
    border-radius: 15px;
    font-size: 1.1em;
    line-height: 1.8;
    border-left: 4px solid #547159;
    margin-bottom: 40px;
    box-shadow: var(--card-shadow);
}

.custom-about-content .section {
    background: var(--card-bg);
    padding: 30px;
    border-radius: 20px;
    backdrop-filter: blur(10px);
    margin-bottom: 30px;
    box-shadow: var(--card-shadow);
}

.custom-about-content .section h3 {
    color: var(--heading-color);
    margin-bottom: 25px;
    font-size: 1.6em;
    text-align: center;
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
}

.custom-about-content .section h3::after {
    content: '';
    position: absolute;
    bottom: -10px;
    left: 50%;
    transform: translateX(-50%);
    width: 80px;
    height: 3px;
    background: linear-gradient(45deg, #547159, #3a4e3f);
    border-radius: 2px;
}

.custom-about-content .section-icon {
    font-size: 1.2em;
}

.custom-about-content .skill-categories {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 25px;
    margin-top: 30px;
}

.custom-about-content .skill-category {
    background: var(--card-bg);
    padding: 20px;
    border-radius: 15px;
    box-shadow: var(--card-shadow);
    transition: transform 0.3s ease;
}

.custom-about-content .skill-category:hover {
    transform: translateY(-5px);
}

.custom-about-content .skill-category h4 {
    color: #547159;
    margin-bottom: 15px;
    font-size: 1.2em;
    display: flex;
    align-items: center;
    gap: 10px;
}

.custom-about-content .skill-category-icon {
    width: 24px;
    height: 24px;
    background: linear-gradient(45deg, #547159, #3a4e3f);
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    color: white;
    font-size: 12px;
    font-weight: bold;
}

.custom-about-content .skill-bars {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.custom-about-content .skill-bar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 8px;
}

.custom-about-content .skill-name {
    font-weight: 500;
    color: var(--text-color);
    font-size: 0.9em;
}

.custom-about-content .skill-level {
    display: flex;
    gap: 3px;
}

.custom-about-content .skill-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: #e0e0e0;
    transition: all 0.3s ease;
}

.custom-about-content .skill-dot.filled {
    background: linear-gradient(45deg, #547159, #3a4e3f);
    transform: scale(1.1);
}

/* Navigation Menu Styles - Clean Minimal Design */
/* .custom-about-content .nav-menu {
    background: transparent;
    padding: 0;
    margin-bottom: 40px;
} */

.custom-about-content .nav-list {
    display: flex;
    flex-wrap: wrap;
    gap: 5px;
    justify-content: center;
    list-style: none;
    margin: 0;
    padding: 0;
    border-bottom: 2px solid rgba(84, 113, 89, 0.1);
    padding-bottom: 15px;
}

.custom-about-content .nav-item {
    background: transparent;
    border-radius: 0;
    overflow: visible;
    transition: all 0.3s ease;
}

.custom-about-content .nav-item:hover {
    transform: none;
}

.custom-about-content .nav-link {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 20px;
    color: var(--text-muted-color);
    text-decoration: none;
    font-weight: 500;
    font-size: 0.95em;
    transition: all 0.3s ease;
    border-bottom: 3px solid transparent;
    position: relative;
}

.custom-about-content .nav-link:hover {
    color: #547159;
    text-decoration: none;
    border-bottom-color: #547159;
    transform: translateY(-2px);
}

.custom-about-content .nav-icon {
    font-size: 1.1em;
}

/* Smooth scroll behavior */
html {
    scroll-behavior: smooth;
}

/* Academic Sections Styles */
.custom-about-content .academic-item {
    background: var(--card-bg);
    padding: 20px;
    border-radius: 10px;
    margin-bottom: 15px;
    border-left: 4px solid #547159;
    transition: transform 0.3s ease;
    box-shadow: var(--card-shadow);
}

.custom-about-content .academic-item:hover {
    transform: translateX(5px);
}

.custom-about-content .academic-item h4 {
    color: var(--heading-color);
    margin-bottom: 8px;
    font-size: 1.1em;
}

.custom-about-content .academic-item .date {
    color: #547159;
    font-weight: 600;
    font-size: 0.9em;
    margin-bottom: 5px;
}

.custom-about-content .academic-item .venue {
    color: var(--text-muted-color);
    font-style: italic;
    margin-bottom: 8px;
}

.custom-about-content .academic-item .description {
    color: var(--text-color);
    line-height: 1.6;
}

.custom-about-content .academic-item .authors {
    color: var(--text-muted-color);
    font-size: 0.95em;
    margin-bottom: 5px;
}

.custom-about-content .news-badge {
    display: inline-block;
    background: linear-gradient(45deg, #e74c3c, #c0392b);
    color: white;
    padding: 2px 8px;
    border-radius: 12px;
    font-size: 0.8em;
    font-weight: bold;
    margin-left: 10px;
}

.custom-about-content .award-badge {
    display: inline-block;
    background: linear-gradient(45deg, #f39c12, #e67e22);
    color: white;
    padding: 2px 8px;
    border-radius: 12px;
    font-size: 0.8em;
    font-weight: bold;
    margin-left: 10px;
}

.custom-about-content .about-text a {
    text-decoration: none;
}

.custom-about-content .about-text a:hover {
    text-decoration: none;
}
.custom-about-content .about-text a {
    text-decoration: none !important;
    color: #007bff;
}

.custom-about-content .about-text a:hover {
    text-decoration: none !important;
    color: #0056b3;
}
@media (max-width: 768px) {
    .custom-about-content .profile-section {
        flex-direction: column;
        text-align: center;
    }
    
    .custom-about-content .about-container {
        padding: 20px;
    }
    
    .custom-about-content .name {
        font-size: 2em;
    }
    
    .custom-about-content .links {
        justify-content: center;
    }

}
.custom-about-content .about-text a,
.custom-about-content .about-text a:link,
.custom-about-content .about-text a:visited,
.custom-about-content .about-text a:hover,
.custom-about-content .about-text a:active {
    text-decoration: none !important;
    border-bottom: none !important;
    color: #007bff !important;
}
</style>

<div class="custom-about-content">
    <div class="about-container">
        <div class="profile-section">
            <div style="width: 250px; height: 250px; border-radius: 50%; background: url('/assets/img/prof.png') center center/cover; border: 5px solid white; background-position: center 20%;"></div>
            
            <div class="profile-info">
                <h1 class="name">Saber Ganjisaffar</h1>
                <p class="position">PhD Candidate</p>
                
                <div class="links">
                    <a href="https://github.com/Archimantix" class="link" data-tooltip="GitHub">
                        <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
                            <path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"/>
                        </svg>
                    </a>
                    <a href="https://www.linkedin.com/in/saber-ganjisaffar/" class="link" data-tooltip="LinkedIn">
                        <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
                            <path d="M20.447 20.452h-3.554v-5.569c0-1.328-.027-3.037-1.852-3.037-1.853 0-2.136 1.445-2.136 2.939v5.667H9.351V9h3.414v1.561h.046c.477-.9 1.637-1.85 3.37-1.85 3.601 0 4.267 2.37 4.267 5.455v6.286zM5.337 7.433c-1.144 0-2.063-.926-2.063-2.065 0-1.138.92-2.063 2.063-2.063 1.14 0 2.064.925 2.064 2.063 0 1.139-.925 2.065-2.064 2.065zm1.782 13.019H3.555V9h3.564v11.452zM22.225 0H1.771C.792 0 0 .774 0 1.729v20.542C0 23.227.792 24 1.771 24h20.451C23.2 24 24 23.227 24 22.271V1.729C24 .774 23.2 0 22.222 0h.003z"/>
                        </svg>
                    </a>
                    <a href="https://scholar.google.com/citations?user=wWO6HXoAAAAJ&hl=en&oi=ao" class="link" data-tooltip="Google Scholar">
                        <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
                            <path d="M5.242 13.769L0 9.5 12 0l12 9.5-5.242 4.269C17.548 11.249 14.978 9.5 12 9.5c-2.977 0-5.548 1.748-6.758 4.269zM12 10a7 7 0 1 0 0 14 7 7 0 0 0 0-14z"/>
                        </svg>
                    </a>
                    <a href="mailto:sganj003@ucr.edu" class="link" data-tooltip="Email">
                        <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
                            <path d="M20 4H4c-1.1 0-1.99.9-1.99 2L2 18c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V6c0-1.1-.9-2-2-2zm0 4l-8 5-8-5V6l8 5 8-5v2z"/>
                        </svg>
                    </a>
                    <a href="/assets/CV.pdf" class="link" data-tooltip="Download CV">
                        <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
                            <path d="M14,2H6A2,2 0 0,0 4,4V20A2,2 0 0,0 6,22H18A2,2 0 0,0 20,20V8L14,2M18,20H6V4H13V9H18V20Z"/>
                        </svg>
                    </a>
                </div>
            </div>
        </div>



        <div class="about-text">
            <p>Hey there! I'm Saber Ganjisaffar, a Ph.D. Candidate in <a href="https://www1.cs.ucr.edu/" target="_blank" rel="noopener noreferrer">CSE</a> at the <a href="https://www.ucr.edu/" target="_blank" rel="noopener noreferrer">University of California, Riverside (UCR)</a>. My research is focused on building secure and efficient computing systems, with a particular focus on hardware-software co-design for security. I delve into computer architecture and microarchitecture, using tools like architectural simulators to explore innovative approaches, identify optimization opportunities, and bring my research ideas to life. I'm also passionate about contributing to open-source simulator projects and collaborating on research that advances the state-of-the-art in system security and efficiency. My work has been published in conferences like <a href="https://iscaconf.org/isca2025/" target="_blank" rel="noopener noreferrer">ISCA</a>, and I'm fortunate to be advised by Professor <a href="https://www.cs.ucr.edu/~nael/" target="_blank" rel="noopener noreferrer">Nael Abu-Ghazaleh</a> and Professor <a href="https://www.cs.ucr.edu/~csong/" target="_blank" rel="noopener noreferrer">Chengyu Song</a>.</p>
            
            
            <p>I'm always open to new collaborations and interesting research opportunities. Feel free to reach out if you'd like to discuss potential projects or just chat about research!</p>
        </div>

        <!-- News Section -->
        <div class="section" id="news">
            <h3><span class="section-icon"></span>News</h3>
            <div class="academic-item">
                <div class="date">July 19, 2025</div>
                <h4>Happy to share that I successfully passed my oral qualifying exam today and have advanced to PhD Candidacy!<span class="news-badge">NEW</span></h4>
                <div class="description"></div>
            </div>
            <div class="academic-item">
                <div class="date">July 10, 2025</div>
                <h4>Excited to be serving on the Artifact Evaluation Committee for MICRO-58 (2025).</h4>
                <div class="description"></div>
            </div>
            <div class="academic-item">
                <div class="date">May 21, 2025</div>
                <h4>Our paper, "SpecASan: Mitigating Transient Execution Attacks Using Speculative Address Sanitization," got accepted to ISCA 2025! 🎉 </h4>
                <div class="description"></div>
            </div>
        </div>

        <!-- Publications Section -->
        <div class="section" id="publications">
            <h3><span class="section-icon"></span>Publications</h3>
            <div class="academic-item">
                <h4><a href="https://dl.acm.org/doi/pdf/10.1145/3695053.3731119" target="_blank" rel="noopener noreferrer">SpecASan: Mitigating Transient Execution Attacks Using Speculative Address Sanitization</a></h4>
                <div class="authors"><strong>S Ganjisaffar</strong>, EM Koruyeh,J Zellmer, HA Esfeden, C Song, NB Abu-Ghazaleh</div>
                <div class="venue">Proceedings of the 52nd Annual International Symposium on Computer Architecture (ISCA 2025)</div>
                <div class="description"></div>

                
            </div>
        </div>

        <!-- Services Section -->
        <div class="section" id="services">
            <h3><span class="section-icon"></span>Services</h3>
            <div class="academic-item">
                <h4>Program Committee Member</h4>
                <div class="venue">MICRO-58 (2025) Artifact Evaluation Committee Member</div>
                <div class="description"></div>
            </div>
            <div class="academic-item">
                <h4>Student Volunteer</h4>
                <div class="venue">ASPLOS 2024 Conference Organizing Team</div>
                <div class="description"></div>
            </div>
        </div>

        <!-- Teaching Section -->
        <div class="section" id="teaching">
            <h3><span class="section-icon"></span>Teaching</h3>
            <div class="academic-item">
                <div class="date">Spring 25, Winter 25, Spring 24</div>
                <h4>Teaching Assistant - CS 010A - Introduction to Computer Science for Science, Mathematics, and Engineering 1 (Beginner C++ Programming)</h4>
                <div class="venue">University of California, Riverside</div>
                <div class="description"></div>
            </div>
            <div class="academic-item">
                <div class="date">Summer 24, Fall 24</div>
                <h4>Teaching Assistant - CS 153 - Design of Operating Systems</h4>
                <div class="venue">University of California, Riverside</div>
                <div class="description"></div>
            </div>
            <div class="academic-item">
                <div class="date">Winter 24</div>
                <h4>Teaching Assistant - CS 010B - Introduction to Computer Science for Science, Mathematics, and Engineering 2 (Intermediate C++ Programming)</h4>
                <div class="venue">University of California, Riverside</div>
                <div class="description"></div>
            </div>
            <div class="academic-item">
                <div class="date">Spring 2020</div>
                <h4>Teaching Assistant - Design of Digital Circuits Course</h4>
                <div class="venue">Shahid Beheshti University</div>
                <div class="description"></div>
            </div>
        </div>

        <!-- Awards Section -->
        <div class="section" id="awards">
            <h3><span class="section-icon"></span>Awards</h3>
            <!-- <div class="academic-item">
                <div class="date">2025</div>
                <h4>Best Student Paper Award <span class="award-badge">GOLD</span></h4>
                <div class="venue">International Conference on AI Applications</div>
                <div class="description">Recognized for outstanding research contribution in the field of neural architecture search.</div>
            </div> -->
            <div class="academic-item">
                <div class="date">2022</div>
                <h4>Dean’s Distinguished Fellowship Award</h4>
                <div class="venue">Awarded by University of California, Riverside (UCR) for Academic year 2022 </div>
                <div class="description"></div>
            </div>
        </div>
        
        <!-- Skills Section -->
        <div class="section" id="skills">
            <h3><span class="section-icon"></span>Skills & Technologies</h3>
            <div class="skill-categories">
                <div class="skill-category">
                    <h4>
                        <span class="skill-category-icon">💻</span>
                        Languages and Frameworks
                    </h4>
                    <div class="skill-bars">
                        <div class="skill-bar">
                            <span class="skill-name">C/C++</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">CUDA</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">OpenCL</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">ARM Assembly</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Python</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Golang</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Verilog</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Chisel</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">SystemC/TLM</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                    </div>
                </div>
                
                <div class="skill-category">
                    <h4>
                        <span class="skill-category-icon">🛠️</span>
                        Tools and Technologies
                    </h4>
                    <div class="skill-bars">
                        <div class="skill-bar">
                            <span class="skill-name">Linux</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                         <div class="skill-bar">
                            <span class="skill-name">Bash</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Git/Github</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">LaTeX</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Xilinx Vivado</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Intel Quartus Prime</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Synopsys Design Compiler</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Intel HLS</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">ModelSim</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                    </div>
                </div>
                
                <div class="skill-category">
                    <h4>
                        <span class="skill-category-icon">🎛</span>
                        Architectural/System Modeling
                    </h4>
                    <div class="skill-bars">
                        <div class="skill-bar">
                            <span class="skill-name">gem5</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">AccelSim (GPGPUSim)</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">MGPUSIM</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Ramulator</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Scarab</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">RocketChip(Chipyard)</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">Verilator</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                        <div class="skill-bar">
                            <span class="skill-name">CACTI</span>
                            <div class="skill-level">
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot filled"></div>
                                <div class="skill-dot"></div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
