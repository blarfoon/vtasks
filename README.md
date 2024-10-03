# VTask

VTask is a flexible and robust asynchronous task management system for Rust, designed to handle complex, interruptible workflows with ease.

## Features:
- Hierarchical task structure with weighted subtasks
- Progress tracking and reporting
- Pausable and cancellable tasks
- Graceful error handling and cleanup
- Interruptible operations within tasks
- Asynchronous execution with synchronous command handling

Ideal for applications requiring structured, long-running operations with fine-grained control and progress monitoring, such as data processing pipelines, background job systems, or multi-step computational tasks.

## Usage
```rust
use vtasks::{VTaskManager, VTaskCommand, Weight, Progress, ProgressUnit, interruptable};
use std::time::Duration;
use tokio::time::timeout;

#[derive(Clone, PartialEq, Eq, Hash, Debug)]
enum TaskName {
    MainTask,
    Subtask1,
    Subtask2,
}

impl Display for TaskName {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            TaskName::MainTask => write!(f, "MainTask"),
            TaskName::Subtask1 => write!(f, "Subtask1"),
            TaskName::Subtask2 => write!(f, "Subtask2"),
        }
    }
}

async fn main() {
    let manager = Arc::new(VTaskManager::new());
    // You first need to create the top level task
    let (id, mut progress_rx) = manager
        .create_task(
            TaskName::MainTask,
            // Define all the subtasks and their weights, even if they might not be used
            vec![
                (TaskName::Subtask1, Weight::Low),
                (TaskName::Subtask2, Weight::High),
            ],
        )
        .await;

    let task = manager.get_task(id).await.unwrap();

    // start async data processing
    let vtask_handle = tokio::spawn(async move {
        let task_clone = task.clone();

        // Here you would start your data processing.
        // Try to keep all your code inside interruptable blocks, and make each interruptable block
        // as small as possible to ensure that the task can be paused and resumed at any point.

        task_clone.run_subtask(&TaskName::Subtask1, |mut context| {
            let task_clone = task_clone.clone();
            async move {
                interruptable(&mut context, || async {
                    task_clone
                        .set_subtask_progress(
                            &TaskName::Subtask1,
                            Some(Progress::new(50.0, 100.0, ProgressUnit::Percentage)),
                        )
                        .await?;
                    Ok(())
                })
                .await?;

                interruptable(&mut context, || async {
                    task_clone
                        .set_subtask_progress(
                            &TaskName::Subtask1,
                            Some(Progress::new(100.0, 100.0, ProgressUnit::Percentage)),
                        )
                        .await?;
                    Ok(())
                })
                .await?;

                Ok::<_, anyhow::Error>(())
            }
        });

        task_clone.run_subtask(&TaskName::Subtask2, |mut context| {
            let task_clone = task_clone.clone();
            async move {
                interruptable(&mut context, || async {
                    task_clone
                        .set_subtask_progress(
                            &TaskName::Subtask2,
                            Some(Progress::new(2.5, 5.0, ProgressUnit::Items)),
                        )
                        .await?;
                    Ok(())
                })
                .await?;

                interruptable(&mut context, || async {
                    task_clone
                        .set_subtask_progress(
                            &TaskName::Subtask2,
                            Some(Progress::new(5.0, 5.0, ProgressUnit::Items)),
                        )
                        .await?;
                    Ok(())
                })
                .await?;

                Ok::<_, anyhow::Error>(())
            }
        });

    });

    match timeout(Duration::from_secs(10), vtask_handle).await {
        Ok(result) => result.expect("VTask panicked"),
        Err(_) => panic!("VTask execution timed out after 10 seconds"),
    }

}
```

## License

VTask is licensed under the MIT License.