
#### CODES BELOW CAN BE USED AS ON _RUST PLAYGROUND_

```
use tokio::select;
use tokio_util::sync::CancellationToken;

#[derive(Debug, Default)]
struct TaskRunTokens{
    group: Option<Vec<(String, CancellationToken)>>,
    single: Option<Vec<(String, CancellationToken)>>
}
impl TaskRunTokens {
    pub fn get_group_token(&mut self, id: &str) -> Option<CancellationToken>{
        let main_token = CancellationToken::new();
        let child_token = main_token.child_token();

        if let Some(group) = &mut self.group {
            if group.iter().any(|(existing_id, _)| existing_id == id) {
                return group.iter()
                    .find(|(existing_id, _)| existing_id == id)
                    .map(|(_, token)| token.child_token());
            }
            group.push((id.to_string(), main_token));
            self.group = Some(group.to_vec());
        } else {
            self.group = Some(vec![(id.to_string(), main_token)]);
        }
        
        Some(child_token)
    }
    
    pub fn get_single_token(&mut self, id: &str) -> Option<CancellationToken> {
        let main_token = CancellationToken::new();
        if let Some(single) = &mut self.single {
            if single.iter().any(|(existing_id, _)| existing_id == id) {
                return single.iter()
                    .find(|(existing_id, _)| existing_id == id)
                    .map(|(_, token)| token.clone());
            }
            single.push((id.to_string(), main_token.clone()));
            self.single = Some(single.to_vec());
        } else {
            self.single = Some(vec![(id.to_string(), main_token.clone())]);
        }
        
        Some(main_token)
    }
    
    pub fn get_main_group_token(&self, id: &str) -> Option<CancellationToken> {
        if let Some(group) = &self.group {
            if let Some((_, main_token)) = group.iter().find(|(existing_id, _)| existing_id == id) {
                return Some(main_token.clone());
            }
        }
        None
    }
    
    pub fn get_main_single_token(&self, id: &str) -> Option<CancellationToken> {
        if let Some(single) = &self.single {
            if let Some((_, main_token)) = single.iter().find(|(existing_id, _)| existing_id == id) {
                return Some(main_token.clone());
            }
        }
        None
    }
}

#[tokio::main]
async fn main() {
    let mut run_tokens = TaskRunTokens::default();
    let mut task_handles: Option<Vec<tokio::task::JoinHandle<String>>> = None;

    for i in 0..5{
        let group_token = run_tokens.get_group_token("group_task_0").unwrap();
        let single_token = run_tokens.get_single_token(format!("single_task_{}", i).as_str()).unwrap();

        // println!("In loop {:?}", i);
        
        let join_handle = tokio::spawn(async move {
            println!("In Task {:?}", i);
            select! {
                _ = group_token.cancelled() => {
                    // println!("Group:{:?} [All Group Tasks Cancelled]", i);
                    format!("cancelled in task:{}", i)
                }
                _ = single_token.cancelled() => {
                    // println!("Task:{:?} [Single Task Cancelled]", i);
                    format!("cancelled in task:{}", i)
                }
                _ = tokio::time::sleep(std::time::Duration::from_secs(10)) => {
                    // println!("user Work finished in task:{:?}", i);
                    format!("user work finished in task:{}", i)
                }
            }
        });
        
        match task_handles.take() {
            Some(mut handles) => {
                handles.push(join_handle);
                task_handles = Some(handles);
            },
            None => {
                task_handles = Some(vec![join_handle]);
            }
        }
    }

    tokio::spawn(async move {
        tokio::time::sleep(std::time::Duration::from_secs(2)).await;
        
        // let main_group_token = run_tokens.get_main_group_token("group_task_0").unwrap();
        // println!("cancelling group token");
        // main_group_token.cancel();
        
        let single_group_token = run_tokens.get_main_single_token("single_task_1").unwrap();
        println!("cancelling single token");
        single_group_token.cancel();
    });
    
    let handles = task_handles.take().unwrap();
    for handle in handles {
        let output = handle.await.expect("error awaiting task");
        println!("task handle output: {:#?}", output);
    }
}

// *************************************** HISTORY ********************************************

// use tokio::task;
// use tokio::time;
// use std::time::Duration;
// // use serde_json::{json, Value};
// // use serde::de::DeserializeOwned;

// // fn get_field<T>(obj: &Value, field: &str, action_type: &str) -> Result<T, String>
// // where
// //     T: DeserializeOwned + Default,
// // {
// //     if action_type != "Element" && action_type != "Spreadsheet" && action_type != "Code" {
// //         match obj.get(field).map(|v| serde_json::from_value(v.clone())) {
// //             Some(Ok(val)) => Ok(val),
// //             _ => Ok(Default::default()), // Use the default value if field is not found or cannot be deserialized
// //         }
// //     } else {
// //         Ok(Default::default()) // Use the default value for these action types as well
// //     }
// // }

// #[tokio::main]
// async fn main() {
//     // let action = json!({
//     //     "actionType": "NewTab",
//     //     "id": "asdasd",
//     //     "nestingLevel": 0,
//     //     "breakingBad": "21",
//     //     "recorded": false
//     // });

//     // let actions: Vec<Value> = vec![action];
    
//     // // println!("actions[0]{:?}", actions[0]);

//     // match get_field::<String>(&actions[0], "id", "Element") {
//     //     Ok(val) => {
//     //         println!("{:?}", val);
//     //     }
//     //     Err(err) => {
//     //         println!("{:?}", err);
//     //     }
//     // };
    
//     // match get_field::<i64>(&actions[0], "nestingLevel", "NewTab") {
//     //     Ok(val) => {
//     //         println!("{:?}", val);
//     //     }
//     //     Err(err) => {
//     //         println!("{:?}", err);
//     //     }
//     // };
    
//     // match get_field::<String>(&actions[0], "breakingBad", "NewTab") {
//     //     Ok(val) => {
//     //         println!("{:?}", val);
//     //     }
//     //     Err(err) => {
//     //         println!("{:?}", err);
//     //     }
//     // };
    
    
    
//     let original_task = task::spawn(async {
//         let _detached_task = task::spawn(async {
//             // Here we sleep to make sure that the first task returns before.
//             time::sleep(Duration::from_millis(10)).await;
//             // This will be called, even though the JoinHandle is dropped.
//             println!("♫ Still alive ♫");
//         });
//     });
    
//     original_task.await.expect("The task being joined has panicked");
//     println!("Original task is joined.");
    
//     // We make sure that the new task has time to run, before the main
//     // task returns.
    
//     time::sleep(Duration::from_millis(1000)).await;
//     println!("THE END.");
// }
```
