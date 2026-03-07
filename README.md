# Awesome Learn 📚

A curated collection of markdown files for learning various technologies and concepts. This repository serves as a central hub for organizing and sharing learning resources across different technology domains.

## 📖 Topics

This repository is organized into the following learning areas:

- **[CI/CD](./ci-cd/)** - Continuous Integration, Continuous Delivery, pipelines, GitHub Actions, GitOps, and DevSecOps
- **[Cloud Computing](./cloud-computing/)** - Cloud fundamentals, AWS, Azure, GCP, networking, serverless, cost optimization, and multi-cloud strategies
- **[Commerce Systems](./commerce-systems/)** - E-commerce platforms, payment systems, and online retail technologies
- **[Databases](./databases/)** - SQL, NoSQL, data modeling, replication, sharding, caching, and database operations
- **[Docker](./docker/)** - Container fundamentals, images, Dockerfiles, networking, storage, Compose, and security
- **[Frontend](./frontend/)** - HTML, CSS, JavaScript, TypeScript, responsive design, state management, and build tools
  - **[Angular](./frontend/angular/)** - Angular framework, components, services, RxJS, signals, and modern Angular patterns
- **[Git](./git/)** - Version control, branching strategies, collaboration workflows, and Git best practices
- **[Infrastructure as Code](./infrastructure-as-code/)** - Terraform, Pulumi, CloudFormation, state management, modules, and multi-cloud IaC
- **[.NET and C#](./dotnet-csharp/)** - .NET ecosystem, C# language, services, breaking changes, best practices, and style guide
- **[Kubernetes](./kubernetes/)** - Container orchestration, cloud-native technologies, and K8s ecosystem
- **[Messaging](./messaging/)** - Message queues, event streaming, Kafka, RabbitMQ, and cloud messaging services
- **[Networking](./networking/)** - Networking fundamentals, IP addressing, routing, DNS, TLS, load balancing, and troubleshooting
- **[Microservices](./microservices/)** - Microservices architecture, design principles, and best practices
- **[Observability](./observability/)** - Monitoring, logging, tracing, and system observability
- **[Security](./security/)** - Application security, authentication, authorization, cryptography, and DevSecOps
- **[System Design](./system-design/)** - Architecture patterns, scalability, distributed systems, caching, load balancing, and reliability
- **[Terminal](./terminal/)** - Terminal tools, tmux, Neovim, shell configuration, and cross-platform setup
- **[TypeScript](./typescript/)** - TypeScript language, type system, design patterns, full-stack development, testing, and tooling

## 🚀 Getting Started

1. Browse through the topic directories to find learning resources
2. Each directory contains a README with an overview and resources
3. Feel free to add your own learning materials following our [contributing guidelines](./CONTRIBUTING.md)

## 🌐 Documentation Site (GitHub Pages + MkDocs)

This repository includes an MkDocs configuration (`mkdocs.yml`) and a GitHub Actions workflow (`.github/workflows/deploy-pages.yml`) to publish documentation to GitHub Pages from the `main` branch.

To keep access limited to collaborators only, keep the repository private and in GitHub Pages settings use the private/access-restricted visibility options available to your plan.

## 📖 Writebook (mdBook)

This repository also includes an [mdBook](https://rust-lang.github.io/mdBook/) configuration in the `writebook/` directory that organizes all markdown content into a navigable, book-style format.

### Build the book locally

```bash
# Install mdBook (https://rust-lang.github.io/mdBook/guide/installation.html)
cargo install mdbook

# Build the book
cd writebook
mdbook build

# Or serve it locally with live reload
mdbook serve --open
```

The generated book will be in `writebook/book/`.

## 🤝 Contributing

We welcome contributions! If you have learning materials, guides, or notes that could help others, please see our [CONTRIBUTING.md](./CONTRIBUTING.md) file for guidelines on how to add them.

## ❓ FAQ

- **Is running GitHub Pages free?**  
  For **public repositories**, yes. For **private repositories**, GitHub Pages availability depends on your plan (typically paid plans), and the published site is still public.

- **If I have to pay for private repo Pages, can I disable it?**  
  Yes. Go to **Repository Settings → Pages** and unpublish/disable the Pages source.

- **How do I disable GitHub Actions for this repository?**  
  Go to **Repository Settings → Actions → General** and set Actions permissions to disable GitHub Actions for this repository.

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details.

## ⭐ Star This Repository

If you find this repository helpful, please consider giving it a star!
